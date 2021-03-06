# M20JS
Yet another js-based tactics rpg.

## Table of Contents  <!-- omit in toc --> 
- [M20JS](#m20js)
  - [Introduction](#introduction)
  - [Plugins and Tools](#plugins-and-tools)
  - [Rule Assumptions](#rule-assumptions)
    - [Encounter Levels](#encounter-levels)
    - [Encounters](#encounters)
    - [Saving throws](#saving-throws)
    - [Distance](#distance)
  - [Rolls](#rolls)
    - [Attack Rolls](#attack-rolls)
  - [Object Model](#object-model)
    - [TingoDB Collection name: game](#tingodb-collection-name-game)
    - [TingoDB Collection name: monsters](#tingodb-collection-name-monsters)
    - [LokiJS Collection name: session](#lokijs-collection-name-session)
  - [Status Effects](#status-effects)
  - [Performance and Scaling](#performance-and-scaling)
## Introduction 
This document will serve as a reference to explain the design of the M20JS application.

It will run on NodeJS, utilizing a nosql database design and websockets.  The nosql database will be stored locally to the server rather than to a full database system like MongoDB.  This is because M20JS isn't meant to be that cumbersome, and by locally storing everything, it decreases the dependencies for anyone wishing to use it.

The intention is for DM's to install with npm and start running games with.  Players simply need to connect to the DM's site with a browser

[▲Top](#m20js)
## Plugins and Tools
* [Dice expression evaluator](https://github.com/dbkang/dice-expression-evaluator), for evaluating statments such as `2d20+5`
* [TingoDB](http://www.tingodb.com/), A locally-stored MongoDB-compatible database.  This is used for more long-term storage, as well as large collections (e.g. chat).
* [LokiJS](http://lokijs.org/#/), for fast in-memory database access.
* [IPify](https://www.ipify.org/), just a quick api call so I can generate some links for DM's to give out to their players.
* [KnockoutJS](http://knockoutjs.com/), for 2-way binding on the front-end.  Proving once and for all that there are better alternatives to AngularJS.
* [Bootstrap](https://getbootstrap.com/), because styling and responsive design is just easier.
* [NodeJS](https://nodejs.org/en/), in case you've never heard of it before.  Download your server here.

[▲Top](#m20js)
## Rule Assumptions
### Encounter Levels
The rules, as written, contradict themselves.  First, they say the encounter level of a fight equals the hit dice of defeated monsters, then *ADD* +1 per doubling of foes.  Then it attempts to clarify this by calling an encounter with 2 Kobolds as EL 2, and an encounter with 4 as EL 3.  

M20JS rejects the contradictory clarification statement.  So, if there are 4 kobolds (kobolds have HD 1), then first it'll determine how many "doublings" of monsters there are (in this case, 2), and then add up the HD of all the monsters (4).  This encounter is EL 6, not EL 3.  

The determination of monster doublings will be determined as follows: 
```javascript
var output = 0;
var enemies = 16;
var enemiesTemp = enemies;
while (true){
    if (enemiesTemp>=2){
    enemiesTemp = Math.floor(enemiesTemp/2);
    output++;
    } else {
        break;
    }
}
return output;
```
But, if you prefer your own version of EL's, then place EL's for any given monster encounter in the `overwriteEL` variable.  

DM's get configuration options on how to cash out EL's.  Once an encounter is completed, the EL's of the encounter are automatically added to the EL pool.  The DM can then cash out the EL's to the players, or wait until the end of the session (or save it til next session).  If you choose auto-division, it'll auto-divide the EL's among the players (if there are any partial EL's as a result of the division, the number is rounded up).  Or, you can distribute stored EL's on a per-player basis from your pool of EL's accumulated.

[▲Top](#m20js)
### Encounters
Encounters can also be set up as AI-controlled.  The monsters' AI is set based on DM discretion.  Stupid AI will attack randomly, or whoever attacked it last.  Semi-smart AI will attack lower hp members before others.  Smart AI will favor mages over melee.  Players attack monsters with standard attacks and spells.  The DM can stop play at any point to take over.

[▲Top](#m20js)
### Saving throws
The rules as written in M20 state that saving throws are skill checks for 2 of the saving throws, and as a mind+level check for will saves.  The human (and half-elf) gets pluses to skill checks.  But does this apply to saving throws?

M20JS allows you to configure this.  The argument can be made that a skill check and a saving throw are 2 different things.  They may be based on the numbers used in a skill check, but they're entirely different outcomes.  For example, in real D&D, a reflex save is not the same thing as an acrobatics check.  A fortitude save is not the same thing as an athletics check.  A will save is not the same thing as an arcana check.

Using the setting `saves_vs_skills : true` will allow for your calculated skill to be used as a saving throw.  This is on by default; if you wish to disable it, set this value to false.

[▲Top](#m20js)
### Distance
Microlite20's rules say you get one action per turn, as in move, fight, whatever.  This means if enemies appear, your first action as a fighter would be "jump in range of the beast so it can hit me".  Or, "hope the enemy's dumb enough to run within striking range so we can hit 'em on *our* turn".

Every DM who's ever run M20 surely does ***not*** run by these rules because that would be asinine.  But, alas, M20JS is configurable.  If you really want to enable that style of play, you may with the config flag of 'oneActionPerTurn'.

The app, by default, assumes that moving and attacking are part of the same turn, but within reason.  There are 3 categories of distance.  Far, in range, and melee.  Melee distance means within 5 feet (your weapon will have a melee range of 5-10 feet).  In range means you can run up *and* hit the monster in the same turn.  This is based on 30 feet worth of movement (consider 5 feet to be a square's worth of movement).  Everyone gets 30' movement speed, unless you, as a DM, wish to implement racial status effects that would enhance or detract from that base value (see: [status effects](#status-effects)).  Far means you'll spend your entire action getting close, but you won't be able to make a melee attack (you can still make a ranged attack or cast a spell).  

[▲Top](#m20js)
## Rolls
### Attack Rolls
```javascript
var attacker = character[x];
if (attacker.mainHand.type==="melee" && attacker.mainHand.hands===1 && !attacker.offHand){
    if (!attacker.useDex){
        var totalStrength = attacker.strength;
        var totalAttackMod = 0;
        for (var i=0; i<attacker.effects.length; ++i){
            totalStrength+=attacker.effects[i].strengthMod;
            totalAttackMod+=attacker.effects[i].attackMod;
            totalAttackMod+=attacker.mainHand.attackMod;
        }
        var strModifier = Math.floor((totalStrength-10)/2);
        var totalMod = totalAttackMod + strModifier;
        var rolls[];
        var multiAttackMinus = 0;
        while (totalMod>=1){
            rolls.push('1d20 + '+ strModifier + ' + ' + totalAttackMod + ' + ' + multiAttackMinus);
            totalMod-=5;
            multiAttackMinus-=5;
        }
        return rolls;
    }
}
```
[▲Top](#m20js)
## Object Model
> Note: "mod" in this case means "modification", not "modifier".  It represents a miscellaneous modifier on a given stat, skill, or roll.

[▲Top](#m20js)
### TingoDB Collection name: game
>Note: encounters have a treasure object containing everything in the encounter that you wish the players to loot upon completion of the encounter.  If you put any weapons or armor on the monsters themselves, the bonuses and malusus granted by the items will be applied to the monster.  Items in a monster's inventory will not get added to the loot pile.  You'll need to add that to the treasure object.
```json
{    
    "gameID" : string,
    "name" : string,  
    "dmName" : string,  
    "dmPrivateID" : string,
    "log" : [ 
        {
            "playerName": string,
            "charName": string,
            "timestamp": number,
            "message": string
        }
    ], 
    "encounters" : [
        {
            "name" : string,
            "status (inactive, active, completed)" : string,
            "monsters" : [
                {
                    "name" : string,
                    "picUrl" : string,
                    "hp" : number,
                    "maxhp" : number,
                    "hitDice (number) " : number,
                    "hpRoll (dX + Y)" : string,
                    "hpavg" : number,
                    "ac" : number,
                    "status" : string,
                    "attacks" : [
                        {
                            "name" : string,
                            "toHit" : number,
                            "damage" : [
                                {
                                    "name (default: base)" : string,
                                    "roll (XdY+Z)" : string
                                }
                            ]
                        }
                    ],
                    "inventory" : {
                        "weapons" : [
                            {
                                "name" : string,
                                "description": string,
                                "type (melee or missile)" : string,
                                "hands (1 or 2, how many hands it takes to hold it)" : number,
                                "attackMod" : number,
                                "damageMod" : number,
                                "damage" : [
                                    {
                                        "name" : string,
                                        "roll (XdY + Z)" : string
                                    }
                                ],
                                "value" : {
                                    "platinum" : number,
                                    "gold" : number,
                                    "electrum" : number,
                                    "silver" : number,
                                    "copper" : number
                                }
                                "globalEffect (does equipping this weapon give you bonuses to anything?  maybe it's cursed and creating negatives?)" : {
                                    "name" : string,
                                    "description" : string,
                                    "alignment (an integer... positive, negative, or 0 for neutral)" : number,
                                    "removeable (boolean)" : string,
                                    "source (weapon)" : string,
                                    "slot (main hand)" : string,
                                    "range (distance of effect)" : number,
                                    "strengthMod" : number,
                                    "dexterityMod" : number,
                                    "mindMod" : number,
                                    "physicalMod" : number,
                                    "subterfugeMod" : number,
                                    "knowledgeMod" : number,
                                    "communicationMod" : number,
                                    "survivalMod" : number,
                                    "savingThrowMod" : number,
                                    "attackMod" : number,
                                    "dualAttackMod" : number,
                                    "acMod" : number,
                                    "skillComboMod (in case there are specific skill+stat combos that will have minuses)" : [
                                        {	
                                            "skill" : string,
                                            "stat" : string,
                                            "mod" : number
                                        }
                                    ]
                                    "damageMod" : number
                                }
                            }
                        ],
                        "wearables" : [
                            {	
                                "name" : string,
                                "description" : string,
                                "alignment (an integer... positive, negative, or 0 for neutral)" : number,
                                "removeable" : boolean,
                                "source (clothing)"  : string,
                                "slot (off hand, chest, left finger, right finger, left bracelet, right bracelet, neck, shoulders, head, underwear, cape, pants, socks, shoes, anklet, left toe ring, right toe ring, nose ring, eyebrow ring, left earring, right earring, lip ring)" : string,
                                "range (distance of effect)" : number,
                                "strengthMod" : number,
                                "dexterityMod" : number,
                                "mindMod" : number,
                                "physicalMod" : number,
                                "subterfugeMod" : number,
                                "knowledgeMod" : number,
                                "communicationMod" : number,
                                "survivalMod" : number,
                                "savingThrowMod" : number,
                                "attackMod" : number,
                                "dualAttackMod" : number,
                                "acMod" : number,
                                "skillComboMod (in case there are specific skill+stat combos that will have minuses)" : 
                                [
                                    {
                                        "skill" : string,
                                        "stat" : string,
                                        "mod" : number
                                    }
                                ], 
                                "damageMod" : number
                            }
                        ]
                    }
                }
            ],
            "treasure" : {
                    "money" : {
                        "platinum" : number,
                        "gold" : number,
                        "electrum" : number,
                        "silver" : number,
                        "copper" : number
                    },
                    "weapons" : [
                        {
                            "name" : string,
                            "description": string,
                            "type (melee or missile)" : string,
                            "hands (1 or 2, how many hands it takes to hold it)" : number,
                            "attackMod" : number,
                            "damageMod" : number,
                            "damage" : [
                                {
                                    "name" : string,
                                    "roll (XdY + Z)" : string
                                }
                            ],
                            "value" : {
                                "platinum" : number,
                                "gold" : number,
                                "electrum" : number,
                                "silver" : number,
                                "copper" : number
                            }
                            "globalEffect (does equipping this weapon give you bonuses to anything?  maybe it's cursed and creating negatives?)" : {
                                "name" : string,
                                "description" : string,
                                "alignment (an integer... positive, negative, or 0 for neutral)" : number,
                                "removeable (boolean)" : string,
                                "source (weapon)" : string,
                                "slot (main hand)" : string,
                                "range (distance of effect)" : number,
                                "strengthMod" : number,
                                "dexterityMod" : number,
                                "mindMod" : number,
                                "physicalMod" : number,
                                "subterfugeMod" : number,
                                "knowledgeMod" : number,
                                "communicationMod" : number,
                                "survivalMod" : number,
                                "savingThrowMod" : number,
                                "attackMod" : number,
                                "dualAttackMod" : number,
                                "acMod" : number,
                                "skillComboMod (in case there are specific skill+stat combos that will have minuses)" : [
                                    {	
                                        "skill" : string,
                                        "stat" : string,
                                        "mod" : number
                                    }
                                ]
                                "damageMod" : number
                            }
                        }
                    ],
                    "wearables" : [
                        {	
                            "name" : string,
                            "description" : string,
                            "alignment (an integer... positive, negative, or 0 for neutral)" : number,
                            "removeable" : boolean,
                            "source (clothing)"  : string,
                            "slot (off hand, chest, left finger, right finger, left bracelet, right bracelet, neck, shoulders, head, underwear, cape, pants, socks, shoes, anklet, left toe ring, right toe ring, nose ring, eyebrow ring, left earring, right earring, lip ring)" : string,
                            "range (distance of effect)" : number,
                            "strengthMod" : number,
                            "dexterityMod" : number,
                            "mindMod" : number,
                            "physicalMod" : number,
                            "subterfugeMod" : number,
                            "knowledgeMod" : number,
                            "communicationMod" : number,
                            "survivalMod" : number,
                            "savingThrowMod" : number,
                            "attackMod" : number,
                            "dualAttackMod" : number,
                            "acMod" : number,
                            "skillComboMod (in case there are specific skill+stat combos that will have minuses)" : 
                            [
                                {
                                    "skill" : string,
                                    "stat" : string,
                                    "mod" : number
                                }
                            ], 
                            "damageMod" : number
                        }
                    ],
                    "misc" : [
                        {	
                            "name" : string,
                            "description" : string,
                            "quantity" : number,
                            "value" : {
                                "platinum" : number,
                                "gold" : number,
                                "electrum" : number,
                                "silver" : number,
                                "copper" : number
                            }
                        }
                    ]
                }
            }
        }
    ]
}
```
[▲Top](#m20js)
### TingoDB Collection name: monsters
>Note: This collection won't have their inventory.  That sort of thing is decided on the fly anyway.  You can add equipment/inventory when you add them to the encounters.
```json
{
    "name" : string,
    "picUrl" : string,
    "hp" : number,
    "maxhp" : number,
    "hitDice (number) " : number,
    "hpRoll (dX + Y)" : string,
    "hpavg" : number,
    "ac" : number,
    "status" : string,
    "attacks" : [
        {
            "name" : string,
            "toHit" : number,
            "damage" : [
                {
                    "name (default: base)" : string,
                    "roll (XdY+Z)" : string
                }
            ]
        }
    ]
}
```
[▲Top](#m20js)
### LokiJS Collection name: session
```json
{
    "gameID" : string,
    "name" : string,
    "dmName" : string,
    "dmPrivateID" : string,
    "encounterLevels" : number, // (this is how many you have accumulated 
                                //	before cashing out)
    "characters" : [
        {
            "name" : string,
            "playerName" : string,
            "privateID" : string,
            "level" : number,
            "hp" : number,
            "maxhp" : number,
            "effect" : [
                {	
                    "name" : string,
                    "description" : string,
                    "alignment (an integer... positive, negative, or 0 for neutral)" : number,
                    "removeable" : boolean,
                    "source (weapon, clothing, aura, racial modifier, class modifier, status effect)"  : string,
                    "slot (string, for weapon and armor types, options are main hand, off hand, both hands, chest, left finger, right finger, left bracelet, right bracelet, neck, shoulders, head, underwear, cape, pants, socks, shoes, anklet, left toe ring, right toe ring, nose ring, eyebrow ring, left earring, right earring, lip ring, aura)" : string,
                    "range (distance of effect)" : number,
                    "strengthMod" : number,
                    "dexterityMod" : number,
                    "mindMod" : number,
                    "physicalMod" : number,
                    "subterfugeMod" : number,
                    "knowledgeMod" : number,
                    "communicationMod" : number,
                    "survivalMod" : number,
                    "savingThrowMod" : number,
                    "attackMod" : number,
                    "dualAttackMod" : number,
                    "acMod" : number,
                    "skillComboMod (in case there are specific skill+stat combos that will have minuses)" [
                        {
                            "skill" : string,
                            "stat" : string,
                            "mod" : number
                        }
                    ], 
                    "damageMod" : number
                }
            ],
            "strength" : number,
            "dexterity" : number,
            "mind" : number,
            "class" : string,
            "race" : string,
            "mainHand" : {
                "name" : string,
                "description": string,
                "type (melee or missile)" : string,
                "hands (1 or 2, how many hands it takes to hold it)" : number,
                "attackMod" : number,
                "damageMod" : number,
                "damage" : [
                    {
                        "name" : string,
                        "roll (XdY + Z)" : string
                    }
                ],
                "value" : {
                    "platinum" : number,
                    "gold" : number,
                    "electrum" : number,
                    "silver" : number,
                    "copper" : number
                }
                "globalEffect (does equipping this weapon give you bonuses to anything?  maybe it's cursed and creating negatives?)" : {
                    "name" : string,
                    "description" : string,
                    "alignment (an integer... positive, negative, or 0 for neutral)" : number,
                    "removeable (boolean)" : string,
                    "source (weapon)" : string,
                    "slot (main hand)" : string,
                    "range (distance of effect)" : number,
                    "strengthMod" : number,
                    "dexterityMod" : number,
                    "mindMod" : number,
                    "physicalMod" : number,
                    "subterfugeMod" : number,
                    "knowledgeMod" : number,
                    "communicationMod" : number,
                    "survivalMod" : number,
                    "savingThrowMod" : number,
                    "attackMod" : number,
                    "dualAttackMod" : number,
                    "acMod" : number,
                    "skillComboMod (in case there are specific skill+stat combos that will have minuses)" : [
                        {	
                            "skill" : string,
                            "stat" : string,
                            "mod" : number
                        }
                    ]
                    "damageMod" : number
                }
            },
            "offHand" : {
                "name" : string,
                "description": string,
                "type (melee or missile)" : string,
                "hands (1 or 2, how many hands it takes to hold it)" : number,
                "attackMod" : number,
                "damageMod" : number,
                "damage" : [
                    {
                        "name" : string,
                        "roll (XdY + Z)" : string
                    }
                ],
                "value" : {
                    "platinum" : number,
                    "gold" : number,
                    "electrum" : number,
                    "silver" : number,
                    "copper" : number
                }
                "globalEffect (does equipping this weapon give you bonuses to anything?  maybe it's cursed and creating negatives?)" : {
                    "name" : string,
                    "description" : string,
                    "alignment (an integer... positive, negative, or 0 for neutral)" : number,
                    "removeable (boolean)" : string,
                    "source (weapon)" : string,
                    "slot (off hand)" : string,
                    "range (distance of effect)" : number,
                    "strengthMod" : number,
                    "dexterityMod" : number,
                    "mindMod" : number,
                    "physicalMod" : number,
                    "subterfugeMod" : number,
                    "knowledgeMod" : number,
                    "communicationMod" : number,
                    "survivalMod" : number,
                    "savingThrowMod" : number,
                    "attackMod" : number,
                    "dualAttackMod" : number,
                    "acMod" : number,
                    "skillComboMod (in case there are specific skill+stat combos that will have minuses)" : [
                        {	
                            "skill" : string,
                            "stat" : string,
                            "mod" : number
                        }
                    ]
                    "damageMod" : number
                }
            },
            "money" : {
                "platinum" : number,
                "gold" : number,
                "electrum" : number,
                "silver" : number,
                "copper" : number
            },
            "inventory" : [
                {
                    "name"
                    "description"
                    "quantity"
                    "value" : {
                        "platinum" : number,
                        "gold" : number,
                        "electrum" : number,
                        "silver" : number,
                        "copper" : number
                    }
                }
            ]
        }
    ],
    "encounter" : { 
        //populate from game.encounters[x]
    } 
}
```
[▲Top](#m20js)
## Status Effects
I chose a certain philosophy about status effects.  A status effect, by definition, is something that will modify you in some way... Maybe it'll add to your armor class.  Maybe it'll add to your damage.  Maybe it'll decrease certain types of rolls for different skill checks or attacks.  To get these modifications, you need to wield a weapon, put on an article of clothing, cast a spell, drink a potion, contract an illness or poison of some sort, or maybe some status effects are just ingrained due to the race you are or the class you picked.

Whenever a roll happens in this game, it's checked against all these status effects.  

As a DM, you're welcome to put any custom status effect on a player as you choose.  A great example is as referenced earlier in this document: movement speed based on race.  An elf would have a bonus to their speed, so you, as a DM, can place an additional bonus to their speed
[▲Top](#m20js)
## Performance and Scaling
M20JS uses LokiJS for its in-game use.  Speed was the most important aspect in choosing a database, so LokiJS really fit the bill here.  

But, as you know, chat logs can get really long.  The best option here is file storage (TingoDB) so you don't run out of memory.  You'll still be able to scroll back in the chat history.  As you scroll up, you'll get access to the previous 50 entries.  Scroll up again, and you'll lose the last 50 entries until you scroll back down again (or click the "bottom" button).  you'll only ever see 50-100 chat messages at any given time.  


*[M20]: Microlite20
*[DM]: Dungeon Master
*[D&D]: Dungeons and Dragons
*[EL]: Encounter Level
