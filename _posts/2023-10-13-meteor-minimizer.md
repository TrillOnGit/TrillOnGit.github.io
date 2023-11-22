---
title: Meteor Minimizer - A Game Jam Game in Retrospect
tags: TeXt
---

Meteor Minimizer was a short game that I made during Game Boy Jam 11 in September of 2023 using the Godot engine.
From the beginning, I sought to implement patterns that Godot supports well - particularly the Observer Pattern.
I found that the pattern was about as strong as I expected - and that was very strong. I employed a singleton (and the singleton pattern) to preload various signals to call or be called on across other classes. I used this for events that, when occuring, various classes would interact with their output - as an example, a meteor being shot by the player's bullet, reducing its hp to 0 would cause a number of events across classes to occur. The point counter would increase, audio would play, we'd check if more meteors needed to be spawned near it, and we would spawn some if we did need to spawn more meteors.

```cs
if (Hp <= 0)
{
    if (Position.Y > 0)
    {
        _autoLoad.EmitAsteroidDestroyed(this);
        _autoLoad.Score += ScoreValue;
    }
    Console.WriteLine(_autoLoad.Score);
    if (Divisions <= 1)
    {
        Split();
    }
    else
    {
        QueueFree();
    }
}
```

Opting to use C# over Godot's GDScript had its ups and downs. For one, I was much more familiar with its paradigms, so it was easier to hit the ground running. However, Godot 4's lack of HTML5 export support meant that I was ultimately unable to create an in-web playable export of my program.

In the designing aspect of the game, participants were given an initial prompt of 'space', which I found could be interpreted in a couple of ways. Of course outer space is one of the first things to jump to one's mind, but so are the ideas of space management. I decided to lean towards outer space, but still encorporate some of those space management ideas in the design of Meteor Minimizer. I allowed meteors to collide with one another, giving the player points, and creating a satisfying cascading effect of meteor debrise destroying meteors continuously, but this strategy for players was only best utilized if they allowed their screen to fill with meteors, taking up as much 'space' as the player felt comfortable with risking. Additionally, opting to use a score limit for progression rather than timed or survival gave the player more room to breathe, allowing them to find the way they want to play and uncover what works best for them.

![Test Stuff](/MeteorMinimizerDemoImage.png)

Movement and the determination of position can be handled in a number of ways in games, and I opted for a relatively simple approach well-suited to a short form game jam, but perhaps not the best suited for a longer-term project. I utilized 2D vectors for velocity and position, alongside individually set and exported values for maximum velocities and acceleration. Each iteration through the Physics Process, we'd check if the maximum velocity has been reached, and if it hasn't, we'd apply acceleration to it, multiplied by delta time. One possible issue here would be, if delta time is fairly large, we could have a noticeable speed increase compared to other objects - for instance, if our Maximum Velocity was 10 units and our current Velocity was 9 - if delta time is 1 then an acceleration of 5 would see our current velocity increase to 14! Well, that would be a problem if we utilized Process instead of Physics Process, which calculates delta time by physics frames rather than the player's frame rate. So problem solved, right? Well, physics process' delta time is still applied at a fixed rate of 1/60th of a second. It is unstated here, but when our meteors explode, we give them a random velocity and position within a range. That random velocity gives us a slight issue - when we're increasing our velocity, one meteor's velocity could be ever so slightly faster than another's - if MeteorA's velocity is 0.001 units faster than MeteorB's, with a Maximum Velocity of 500, MeteorA could have a Maximum Velocity of 500.001 to MeeteorB's 500! Its... admittedly hardly noticable here, but if we wanted to use smaller units, it could be a bigger issue. A more traditional approach would be to just continually apply acceleration and clamp the velocity to be at most the Maximum Velocity. There may technically be an incredibly slightly raised technical load doing so, but its really just better than my approach here in every other way.

Another point of improvement I would look to is how I handled the game's walls - the idea is that when a meteor collides with the left or right edge of the screen, it will bounce back in. I did this by having each meteor check if it was colliding with an area - areas which were placed on the left and right of the screen. If the meteor was colliding with one of those areas, it would reverse its X velocity. Now, the problem we have is that when a meteor is destroyed, it can spawn a meteor outside the bounds - that meteor would reverse its X velocity which will usually direct it back into the play area, but we can actually from our pseudo-random range get nearly 0 for our x Velocity, which would just let the meteor drift down just outside of the player's view.

```cs
private void OnBarrierEntered(AsteroidBarrier asteroidBarrier)
{
    switch (asteroidBarrier.Direct)
    {
        case AsteroidBarrier.BounceDirection.LeftAndRight:
            Velocity.X *= -1;
            break;
        case AsteroidBarrier.BounceDirection.UpAndDown:
            Velocity.Y *= -1;
            break;
        default:
            throw new ArgumentOutOfRangeException();
    }
    Console.WriteLine("Asteroid bounced");
}
```

The solution I used to prevent this was just to free the queue of any meteor that would fall outside the bounds of the screen - meteors still bounce this way, but if they're spawned outside, then they are harmlessly removed.

Outside of that, there were a couple of areas that could have been fleshed out more with time. It did not include the functional score saving feature that I had planned for. I also felt that music was on the lower end of my priorities for this project, ultimately meaning that while SFX were implemented, music was not.

In conclusion, I think that I learned a fair bit about Godot the variety of approaches' positives and negatives to game development. It was a bit rough around the edges in the end, but its still something that I very much enjoyed and took a lot away from! I'm looking forward to adding more game jam games to my list of projects going forward.
