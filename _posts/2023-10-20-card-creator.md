---
title: Card Creator - A Custom Web App in Retrospect
tags: TeXt
---

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
