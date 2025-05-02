---
title: Project T - An Ongoing Project
tags: TeXt
---

Early on in its development, I joined a game development project, forming a two-person team developing a top-down,
grid-based tactics RPG. We divided our work such that I would pick up the work on the back-end and game design,
while my partner would oversee the front-end and writing. We're still developing this project but I thought it
would be a good idea to go into here.

Early on, we opted to integrate LDTK into our development environment, using its pure JSON data to assign values for
any relevant level design information.

However, to first familiarize myself with the codebase, my first contribution was to implement a mechanic called 'unit exhaustion'.
But I'll begin with an overview of the project itself so that that actually makes sense.

<h2>Project T - Overview</h2>

<!-- Insert image of the project -->![](/ProjectT.png)

Fundamentally, Project T is a game where the player controls a group of mercenaries, navigating their units through
battles as the shifting politics of the land guides their adventure through greater and greater peril. But that doesn't
say much about the medium itself, does it? As a top-down, grid-based system, the player sees a battlefield from above
and controls their units by selecting them one at a time. When all of the player's units have moved and possibly acted,
the player's turn ends and the AI controlled units begin acting one at a time.

The original goal of this project was simply to create a framework to build top-down turn based tactics games off of,
but as the project developed it quickly shifted into its own game project. So, perhaps keeping to its roots, the project
shifted to a focus on iterative design to follow the fun.

<h2>LDTK Integration</h2>

LDTK, or Level Developer's Toolkit, is a development tool that you can pass 2D sprites, or values into and use to create
custom maps in a square grid system. You can tie various values to entities you make in its display, and whenever you save,
it modifies a json file that its visual display is referencing. However, we also can reference the data from its pure JSON
file to construct elements of the game on our own end.

```rust
for layer in level.layer_instances.iter().flatten() {
    for e in layer.entity_instances.iter() {
        if e.identifier != "Unit" {
            continue;
        }
        let mut unit = Unit::default();
        // ldtk uses y-down, we use y-up
        let pos = IVec2::new(e.grid.x, layer.c_hei - e.grid.y - 1);
        for f in e.field_instances.iter() {
            match (f.identifier.as_str(), &f.value) {
                ("max_hp", &FieldValue::Int(Some(hp))) => {
                    unit.hp = hp as usize;
                    unit.max_hp = hp as usize;
                }
                ("max_mana", &FieldValue::Int(Some(mana))) => {
                    unit.mana = mana;
                    unit.max_mana = mana;
                }
            }
        }
    }
}
```

For example, in this function, we take in values from our LDTK level JSON file using Bevy ECS LDTK API. These values are
passed into custom Units we've made from a Struct made to store our relevant unit data. Then our Rust code references the
values stored by our Unit Structs.

But in LDTK since we're passing through visual elements like sprites, the LDTK construction (and thus our code's reference)
actually serves as a very good visual representation of the initial level design state.

<!-- Insert Image of LDTK and maybe the game Here -->![](/LDTKProject.png)

This allows us to make changes quickly and efficiently.

<h2>Abilities and Structured Development</h2>

Through prototyping and discussion, our project shifted to an ever growing focus on 'abilities'. Essentially, a unit would
have there'd be a list of effects mostly unique to a given unit to create a larger variety of situations. Essentially,
the goal was to create more options for a given character at any point. But, with this, the development demanded a solid,
efficient system for ability creation to allow faster development. It also asked for a pretty flexible system.

The system that I would come to work with to build abilities was a Command Builder system constructed with coroutines. So,
lets actually go through an ability that I wrote - Shove. The intent of this ability was, when a unit used it, they could
target an adjacent character. Then, that adjacent character would get moved one space directly away from the unit, so long
as there isn't another unit in that position or a wall in their way.

```rs
//If the StackEvent Shove is called
StackEvent::Shove { unit_id, target } => {
    //Get a reference to a unit from its id (if it doesn't exist, break out)
    let Some(unit) = self.get_unit(unit_id) else {
        return;
    };
    //Get a reference to the unit being targeted
    let Some(target_unit) = self.get_unit(target) else {
        return;
    };
    //Find the new position for the unit (the locations outside of the map are treated as walls)
    let mut pos = (target_unit.pos - unit.pos) + target_unit.pos;
    //If the new position has a unit or a wall, their end destination is just their current position.
    if self.unit_map.contains_key(&pos) || self.get_terrain(&pos) == Terrain::Wall {
        pos = target_unit.pos;
    }
    //Then, the move event is pushed onto our 'stack of events' with the target position.
    self.stack.push(StackEvent::UnitMove {
        unit_id: target,
        to: pos,
    });
}
```

There's a lot of things that are hidden here, such as how it gets called. But this is what is actually called (plus some comments
for clarity). For the command builder itself, it works by taking a Question, such as - after a movement is selected - 'What
action choice has the player selected?'. Then it builds an answer from that question, for instance, if the answer is 'Shove',
the answer will pass to our Ability Builder the ability (Shove) that is being used.

The ability builder, then has more assumptions about what a given ability needs. For instance, Shove needs a target unit, so we'll
get that here and that will be passed later into our StackEvent for Shove.

```rs
impl CoroutineFn for ShoveCoroutine {
    type Output = Command;
    //To cover the variety of asks abilities could have, execute uses contextual types and returns an option of the output
    fn execute<'a>(&self, ctx: &mut CoroutineContext<'a>) -> Option<Self::Output> {
        //Because we'll need to get a unit, for Shove, get_unit will take in the targetable units and the end destination of movement
        let unit = ctx.get_unit(|| {
            (
                //get the units in minimum range 1, maximum range 1 measured from the destination end point
                ctx.game.get_ability_targetable_units_from(
                    self.unit_id,
                    self.movement_path.last().copied(),
                    (1, 1),
                ),
                //get the end movement point for the movement used before casting the ability
                Some(vec![ctx
                    .game
                    .get_path_end(self.unit_id, &self.movement_path)]),
            )
        })?;
        //Finally, we have the information necessary for the Command. This is used by our game to apply the action.
        Some(Command {
            unit_id: self.unit_id,
            movement_path: self.movement_path.clone(),
            action: crate::Action::Ability(AbilityAction::Shove { target: *unit }),
        })
    }
}
```

<h2>Design: RNG and Tactical Identity</h2>

As a genre, 'tactics games' and 'puzzle games' seem to just be differentiated by tactics having the presence of unforseen
situations creating a new element that the player needs to adapt to. So, to many, RNG has been accepted as a given for
the genre. But in my experience, many iterations of RNG (ex. A unit may have a 10% chance of their action failing for example)
lead to negative player experiences - I'd guess that tactics games are turned off or reset midway through a play experience
more than any other genre of game.

Thus, we have to ask, can we remove RNG entirely while maintaining an identity of a tactics game, and how important is that identity
to us? Well, through prototyping and playtesting, I think that we can fairly easily maintain a more technically correct
definition of a tactics game identity - a game which places emphasis on short term goals with a variety of options for the
player to take. To entrenched tactics game players, they'll likely parse a more deterministic game as being a series of puzzles
and that's alright. Even though they are our core audience, we'd like to use the allure of determinism to draw them in, and the
chaos of a wide variety of actions for them to shape their own tactical problems to solve.

Ultimately, our approach is to view RNG as an old method of injecting chaos into a player's playthrough - a consistent way to
make every experience feel unique. We want to inject enough unique opportunities to a player that they can act as our source of
chaos for their own play. This methodology does have its downsides - its asks me, the designer, to create varied, chaos-inducing
options for the player in large enough amounts that the player can feel like almost no other player is having the same experience.
And of course those chaos-inducing options have to be implemented. Ultimately, this methodology has shaped the structure of the
project and I'm excited to see where it goes and just how much it asks.
