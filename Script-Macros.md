## Script Macros
Script Macros in Foundry use the underlying [JavaScript API](https://foundryvtt.com/api) to execute various commands. The API respects the permissions of the user executing the macro, so players cannot delete your big bad evil guy for example.

A community-contributed repository of Script Macros can be found here: https://github.com/foundry-vtt-community/macros

Some examples of using script macros follows.

### Examples
#### Random Table Roll 

```js
const table = game.tables.entities.find(t => t.name === "MY TABLE NAME");
table.draw();
```

Replace the table name with the one you want to use.

#### Rolling perception (in gm mode) for selected tokens or active players if no token is selected. For dnd5e
```js
let actors = [];
// getting all actors of selected tokens
for(let id in canvas.tokens._controlled){
  actors.push(canvas.tokens._controlled[id].actor);
}
// if there are no selected tokens, get the actors the actors that represent active players
if(actors.length<1) {
  for(let id in game.users.entities) {
    if(game.users.entities[id].active){
	  if(game.users.entities[id].character !== null) {
	    actors.push(game.users.entities[id].character)
  	  }			
	}
  }
}

// roll for every actor
let messageContent = '';
for(let actor of actors) {
  let modifier = actor.data.data.skills.prc.mod; // this is total bonus for perception (abilitie mod + proficiency)
  let result = new Roll(`1d20+${modifier}`).roll().total; // rolling the formula
  messageContent += `${actor.name} rolled <b>${result}</b> for perception<br>`; // creating the output string
}

// create the message
if(messageContent !== '') {
  let chatData = {
    user: game.user._id,
    speaker: ChatMessage.getSpeaker(),
    content: messageContent,
    whisper: game.users.entities.filter(u => u.isGM).map(u => u._id)
  };
  ChatMessage.create(chatData, {});
}
```
You can edit `${actor.name} rolled <b>${result}</b> for perception<br>` to change the displayed text. Written by @Felix#6196, if there are any questions feel free to send a message.

#### Pulls Passive Perception for all of your players and sends it to you as a private message. For dnd5e
```js
let actors = game.actors.entities.filter(e=> e.data.type==='character');


// pull each player's passive perception
let messageContent = '';
let messageHeader = '<b>Passive Perception</b><br>';
for(let actor of actors) {
  let modifier = actor.data.data.skills.prc.mod; // this is total bonus for perception (abilitie mod + proficiency)
  let result = 10 + modifier; // this gives the passive perception
  messageContent += `${actor.name} <b>${result}</b><br>`; // creating the output string
}

// create the message
if(messageContent !== '') {
  let chatData = {
    user: game.user._id,
    speaker: ChatMessage.getSpeaker(),
    content: messageHeader + messageContent,
    whisper: game.users.entities.filter(u => u.isGM).map(u => u._id)
  };
  ChatMessage.create(chatData, {});
}
```
Written by @Erogroth#7134 using @Felix#6196's code as a base along with some help from him as well.

### Rage toggle for inhabited character
This rather complex macro enables toggling the barbarian rage for a character. It adds damage resistance for bludgeoning, piercing and slashing, and adds the rage bonus (as determined by the barbarian class level) to the special traits.
The character in question has to have a class called "Barbarian" with the specific level, since that is what's used to determin the damage bonus
Written by Felix#6196, feel free to message if there are any questions.
```js
let macroActor = game.user.character;
let barb = macroActor.items.find(i => i.name === 'Barbarian');
if (barb !== undefined) {
    let chatMsg = '';
    let enabled = false;
    if (macroActor.data.flags.rageMacro !== null && macroActor.data.flags.rageMacro !== undefined) {
        enabled = true;
    }
    if (enabled) {
        chatMsg = `${macroActor.name} is no longer raging.`;

        // reset resistances
        let obj = {};
        obj['flags.rageMacro'] = null;
        obj['data.traits.dr'] = macroActor.data.flags.rageMacro.oldResistances;
        obj['data.bonuses.mwak.damage'] = macroActor.data.flags.rageMacro.oldMwak;
        macroActor.update(obj);

        // optional extra step to toggle rage icon on token
        let token = canvas.tokens.ownedTokens.find(t => t.actor.id === macroActor.id);
        token.toggleEffect('worlds/odyssey/images/playerfiles/icons/rage.svg');
    } else {
        chatMsg = `${macroActor.name} is raging.`;

        // calculate rage damage
        let barblvl = barb.data.data.levels;
        let ragedmg = 2 + Math.floor(barblvl / 9) - (barblvl === 8 ? 1 : 0);

        // update actor
        let obj = {};
        obj['flags.rageMacro.enabled'] = true;
        obj['flags.rageMacro.oldResistances'] = JSON.parse(JSON.stringify(macroActor.data.data.traits.dr));
        obj['flags.rageMacro.oldMwak'] = macroActor.data.data.bonuses.mwak.damage;

        let newResistance = macroActor.data.data.traits.dr;
        if (newResistance.value.indexOf('bludgeoning') === -1) newResistance.value.push('bludgeoning');
        if (newResistance.value.indexOf('piercing') === -1) newResistance.value.push('piercing');
        if (newResistance.value.indexOf('slashing') === -1) newResistance.value.push('slashing');
        obj['data.traits.dr'] = newResistance;
        obj['data.bonuses.mwak.damage'] = ragedmg;

        macroActor.update(obj);

        // optional extra step to toggle rage icon on token
        let token = canvas.tokens.ownedTokens.find(t => t.actor.id === macroActor.id);
        token.toggleEffect('worlds/odyssey/images/playerfiles/icons/rage.svg');
        token.update({ img: 'worlds/odyssey/images/playerfiles/artagantokenANGERY.png' });

        // remove charge
        let rageItem = macroActor.items.find(i => i.name === 'Rage');
        if (rageItem) {
            rageItem.update({ 'data.uses.value': rageItem.data.data.uses.value - 1 })
        }
    }
    // write to chat
    let chatData = {
        user: game.user._id,
        speaker: ChatMessage.getSpeaker(),
        content: chatMsg
    };
    ChatMessage.create(chatData, {});
}
```

### Consume various resources. 

This was actually written as a module that provides the dailyGrind function to be called from a macro with
```js
MacroModule.dailyGrind(actor, [["ration1", 1], ["ration2", 2]])
```

As a standalone macro it looks like:

```js
// Consume items from inventory
let consumption = [["ration1", 1], ["ration2", 2]];
let updates = [];
let consumed = "";
for (let [name, quantity] of consumption) {
  let item = actor.items.find(i=> i.name===name);
  if (item.data.data.quantity < quantity) {
    ui.notifications.warn(`${game.user.name} not enough ${name} remaining`);
    consumed += `Only had ${item.data.data.quantity} ${name} but needed ${quantity} - death ensues<br>`;
  } else {
    updates.push({"_id": item._id, "data.quantity": item.data.data.quantity - quantity});
    consumed += `Consumed ${quantity} ${item.data.name} and has ${item.data.data.quantity - quantity} left<br>`;
  }
}

if (updates.length > 0) {
  actor.updateManyEmbeddedEntities("OwnedItem", updates);
}
ChatMessage.create({
  user: game.user._id,
speaker: { actor: actor, alias: actor.name },
  content: consumed,
  type: CONST.CHAT_MESSAGE_TYPES.OTHER
});
```

### Rolling a skill check for DnD 5e

Here are a few simple scripts useful for rolling skill checks using the dnd5e system.

```js
// This would roll an athletics skill check for the players character. 
// Just replace 'ath' with the appropriate skill abbreviation for the
// skill you'd like to roll.
character.rollSkill('ath');

// This would be used if a player controls more than one character.
// It would roll the check for which ever token the player has selected.
actor.rollSkill('ath');
```

List of skill abbreviations:

* "acr" => Acrobatics
* "ani" => Animal Handling
* "arc" => Arcana
* "ath" => Athletics
* "dec" => Deception
* "his" => History
* "ins" => Insight
* "itm" => Intimidation
* "inv" => Investigation
* "med" => Medicine
* "nat" => Nature
* "prc" => Perception
* "prf" => Performance
* "per" => Persuasion
* "rel" => Religion
* "slt" => Sleight of Hand
* "ste" => Stealth
* "sur" => Survival

### Rolling an attribute check for DnD 5e

These are examples for rolling an attribute check.

```js
// This would pop up a dialog asking if you'd like
// to roll a Strength Test or a Strength Save.
// Just replace "str" with the appropriate ability
// abbreviation.
character.rollAbility("str");

// or this for a player that is controlling multiple
// tokens.
actor.rollAbility("str");

// This would skip the dialog and roll an ability test
character.rollAbilityTest("str");

// And this would skip the dialog and roll an ability save
character.rollAbilitySave("str");
```

List of ability abbreviations:

* "str" => Strength
* "dex" => Dexterity
* "con" => Constitution
* "int" => Intelligence
* "wis" => Wisdom
* "cha" => Charisma

### Play Audio
This is a direct way to play audio.
```js
AudioHelper.play({src: "audio/SFX/Fire arrow.mp3", volume: 0.8, autoplay: true, loop: false}, true);
```

There is a way to play from a playlist which I will add once I have figured out what I, or it is doing wrong but it's supposed to go along the lines of:

```js
const playlist = game.playlists.entities.find(p => p.name === "SFX");
const sound = playlist.sounds.find(s => s.name === "chest open");
playlist.playSound(sound);
```