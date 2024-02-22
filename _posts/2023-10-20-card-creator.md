---
title: Card Creator - A Custom Web App in Retrospect
tags: TeXt
---

![Testing Stuff](/assets/images/FrontPageImage.png)

In August 2023, I worked in a team to develop a custom web application - our goal was to create a tool that would allow its users to input various properties to create their own cards inspired by the popular trading card game Magic the Gathering. We set out to utilize an MVC (Model - View - Control) pattern, and did so while utilizing Razor components - this approach lead us to believe that ultimately a Razor or MVVM pattern approach would have worked well for what we aimed to achieve.

I primarily oversaw backend work (data handling, processing user input, CRUD functionality, unit testing, etc.) and version control. For this project, we opted to use SQLite for our database with Dapper as our ORM to allow us to code in SQL within our models to then act on our database.

```cs
public async Task CreateCard(Card card)
{
    await _conn.ExecuteAsync(
        "INSERT INTO CardData (CardName, userID, CardImg, CARDTEXT, CardFlavorText, CardIllustrator, cardRARITY, " +
        "cardTYPE, cardSUBTYPE, cardPOWER, cardTOUGHNESS, ISLEGENDARY, W, U, B, R, G, C) VALUES (@name, @userId, " +
        "@cardImg, @text, @flavor, @illustrator, @rarity, @type, @subType, @power, @toughness, @isLegendary, " +
        "@white, @blue, @black, @red, @green, @colorless);",
        new
        {
            name = card.Name, text = card.CardText, userId = card.UserId, cardImg = card.CardImg,
            flavor = card.CardFlavorText, illustrator = card.Illustrator,
            rarity = card.Rarity, type = card.Type, subType = card.SubType, power = card.Power,
            toughness = card.Toughness, isLegendary = card.IsLegendary, white = card.CardCost.White,
            blue = card.CardCost.Blue, black = card.CardCost.Black, red = card.CardCost.Red,
            green = card.CardCost.Green, colorless = card.CardCost.Colorless
        });
}
```

The core of my work on this project was to oversee the processes of storing the user's input, then taking those stored values from the database, and using that data to display the result of the user's input (the card generated from the saved data). I did this by first taking the user input from a CSS table that they filled out, and saving it into our SQLite database. Then, when querying for the display of the card, the saved data would be passed into a class that would handle the data, converting it into more useful values for the visual display. Additionally, we used Razor page Authentication to associate cards with specific user IDs that were tied to passwords and emails - this allowed us to utilize a central database where users could access their account's data across devices without seeing other user information.

Much of the backend work was done within the card class, and having fleshed that out I thought I'd go through some of the code. So, the obvious values we needed (if you know about the rules of Magic: The Gathering) are the colors for the card, rarity, the illustrator, the name, the type, the sub-type, power, toughness, description, and flavour text. But, we also need some other things - we need the user's id to be tied to the card, we need to save the image they use, we need to store an ID for the card itself, and we should store if it is legendary, but we also need to store what combination of colors on the card is going to result in the card's inner border changing color and its outer frame changing color.

These items being dependent on certain color values makes storing these values more complex than storing simple strings, we need to store colors as binaries, and then after storing those we still need to give back those values in a string format. So, we utilize polymorphism since one of these ideas is commonly used in converting to strings, naming a method 'ToString' and another we name the conventional 'Parse'. These were defined in our ManaCost class along with the values for each color we can have.

This code checks if there are values of white, blue, black, etc. and appends the corresponding character to the string if it is there. Then, if the string is not empty it returns it, otherwise, it returns "0".

```cs
public override string ToString()
{
    var convertString = $"{(Colorless > 0 ? Colorless : "")}";
    uint[] workArr = { White, Blue, Black, Red, Green };
    char[] charArr = { 'W', 'U', 'B', 'R', 'G' };

    for (int i = 0; i < workArr.Length; i++)
    {
        if (workArr[i] != 0)
        {
            convertString += new string(Enumerable.Repeat(charArr[i], (int)workArr[i]).ToArray());
        }
    }

    return convertString != "" ? convertString : "0"; //If there's no mana cost, return 0 instead of nothing.
}
```

This code references a helper class - an internal, static class for Card. It contains a dictionary that employs functional programming to cleanly correlate its key to a function, which adds to its corresponding values:

```cs
public static ManaCost Parse(string stringInput)
{
    var convertMc = new ManaCost();
    foreach (char elem in stringInput)
    {
        if (ColorHelpers.ColorCountByChar.Keys.Contains(elem))
        {
            var action = ColorHelpers.ColorCountByChar[elem];
            action(convertMc);
        }
    }
    convertMc.Colorless = CardCostHelper.GetColorlessMana(stringInput);

    return convertMc;
}
```

```cs
internal static readonly IDictionary<char, Action<ManaCost>> ColorCountByChar = new Dictionary<char, Action<ManaCost>>()
{
    { 'W', c => c.White += 1  },
    { 'U', c => c.Blue += 1  },
    { 'B', c => c.Black += 1  },
    { 'R', c => c.Red += 1 },
    { 'G', c => c.Green += 1  }
};
```

Then it uses this simple function to handle its colorless values!

```cs
public static uint GetColorlessMana(string cardManaCost)
{
    var storageString = cardManaCost.Where(char.IsDigit).Aggregate("0", (current, t) => current + t);
    return uint.Parse(storageString);
}
```

For our frame and inner colors, we need to know if we're using a small combination of colors, or if there are too many colors then we'll instead use a gold color. So, how can we check if these are true or false? Well, I utilized a flag enum and some bit fiddling like so to store them:

```cs
[Flags]
public enum AdjustingColor
{
    Colorless = 0,
    White = 1,
    Blue = 1 << 1,
    Black = 1 << 2,
    Red = 1 << 3,
    Green = 1 << 4,
    Gold = 1 << 5
}
```

For actually determining these enum values, we do so by checking for colors in the player's card one at a time by looking for a character corresponding to each color, setting the corresponding bit to 1 and then if enough are present, setting the gold value to 1 (or in other words, True). We do a very similar thing for the outer frame, using 3 colors instead of 2 to determine if we should use gold.

```cs
public static AdjustingColor GetInnerColor(ManaCost inputCost)
{
    var colorCount = 0;
    var innerColor = AdjustingColor.Colorless;
    var manaCostArr = new[] { inputCost.White, inputCost.Blue, inputCost.Black, inputCost.Red, inputCost.Green };
    for (var i = 0; i < manaCostArr.Length; i++)
    {
        if (manaCostArr[i] >= 1)
        {
            colorCount++;
            innerColor |= (AdjustingColor)(1<<i);
        }
    }
    return colorCount >= 2 ? AdjustingColor.Gold : innerColor;
}
```

To verify that these processes worked while in development, I employed unit testing methodology and the xUnit.net framework. I utilized inline data and member data for varying cases. Because each of the tests was performed on methods, I used theory instead of fact tests. I created tests mostly for the card class, testing the ability to determine the total cost of a card from a string, the ability to create a string from enum values, the ability to determine inner and frame colors, and the ability to determine the amount of colorless resources. All functionality that the Card and ManaCost classes handle.

```cs
public static TheoryData<ManaCost, string> ManaCostToStringTestData => new() {
    { new ManaCost { Colorless = 1}, "1" },
    { new ManaCost { Colorless = 1, White = 2 }, "1WW" },
    { new ManaCost { White = 1, Blue = 2, Red = 1}, "WUUR"},
    { new ManaCost { }, "0"}
};
[Theory]
[MemberData(nameof(ManaCostToStringTestData))]
public void ManaCostToStringTest(ManaCost costInput, string expected)
{
    var actual = costInput.ToString();

    Assert.Equal(expected, actual);
}
```

While working on this project I employed asynchronous processing to allow multiple processes to be handled at once. In all reality, this project likely did not significantly benefit from the use of asynchronicity. The processes here are generally likely to be simple unless we have a large dataset to display from our CRUD functionality.

While the entirety of CRUD functionality is handled asynchronously, this is an example snippet of async usage:

```cs
public async Task<IActionResult> UpdateCardToDatabase(Card card, IFormFile img)
{
    if (img is { Length: > 0 })
    {
        card.CardImg = new byte[img.Length];
        await img.OpenReadStream().ReadAsync(card.CardImg, 0, (int)img.Length);
    }
    await _repo.UpdateCard(card);

    return RedirectToAction("ViewCard", new { id = card.CardId });
}
```

Razor Pages allowed for implementing a simple authentication feature to work well. Users input an email and password, and in the back end, we tied these to their user ID which was then saved with each card a user made. This does have a couple of security issues that were pushed aside to get the project out more quickly. The functionality of users only having direct access to their own created cards was still very appealing for this project.

![Authentication Image](/assets/images/AuthenticationImage.png)

I also worked on the library display for user cards, providing enough information for users to navigate effectively between cards without the experience being overwhelming. This was primarily just taking saved user data from our SQL server and providing it to the user alongside a clickable name corresponding to their card that then redirects them to a page where they can make edits to their card.

![Library Image](/assets/images/LibraryImage.png)

In conclusion, this was an extensive project that helped to sharpen a lot of my skills. It is a project that has a lot of room to be worked further on, but still allowed me to become a lot more comfortable with SQL, data handling, and the process of website creation.
