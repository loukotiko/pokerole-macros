# Helpful Macros for Pokérole system for Foundry

## [GM] Pokéfill : Auto-fill Pokémon sheets
**Macro type :** script
**Description :** Automatically fills attributes of a Pokémon, based on Moves it can learn.
```js
const skillsRankData = {
  "starter": {attributes:0,skills:5,social:0, max: 1},
  "beginner": {attributes:2,skills:9,social:2, max: 2},
  "amateur": {attributes:4,skills:12,social:4, max: 3},
  "ace": {attributes:6,skills:14,social:6, max: 4},
  "pro": {attributes:8,skills:15,social:8, max: 5},
  "master": {attributes:8,skills:15,social:8, max: 5},
  "champion": {attributes:8,skills:15,social:8, max: 5},
};

const rankValue = {
  "starter": 1,
  "beginner": 2,
  "amateur": 3,
  "ace": 4,
  "pro": 5,
  "master": 6,
  "champion": 7,
};

const defaultStat = {min: 0, max: 5, value: 0};

const getType = (str) => str === 'will' ? 'other' : ['strength', 'dexterity', 'vitality', 'special', 'insight'].includes(str) ? 'attributes' : ['tough', 'cool', 'beauty', 'cute', 'clever'].includes(str) ? 'social' : 'skills';

const sum = (obj) => Object.values(obj).reduce((sum,value) => sum+value, 0);

function choseRandomWeighted(weighted) {
  if(!Object.entries(weighted).length) return;
  const thresholds = Object.entries(weighted).reduce((thresholds, [key, weight], i) => {
    thresholds[i] = {key, weight: weight + (thresholds[i-1]?.weight||0)};
    return thresholds;
  }, []);

  let random = Math.random() * thresholds[thresholds.length - 1].weight;
  
  let i;
  for (i = 0; i < thresholds.length; i++) {
    if (thresholds[i].weight > random) break;
  }
  return thresholds[i].key;
}

async function handleDrop(actor, sheet, {rank, type}) {
  if(type != "PkmnFill") return;
  const actualRank = actor.rank;

  const moves = actor.items.filter((value) => value.type === 'move');
  const weights = moves.reduce((weights, value) => {
    const {system: {accMod1, accMod2, dmgMod}} = value;

    let weight = 1;
    if(value.system.type !== 'none') {
      weight+=2;
      if(rankValue[value.system.rank] <= rankValue[rank]) weight+=2;
    }
    if(accMod1) weights[getType(accMod1)][accMod1] = (weights[getType(accMod1)][accMod1]||0) + weight;
    if(accMod2) weights[getType(accMod2)][accMod2] = (weights[getType(accMod2)][accMod2]||0) + weight;
    if(dmgMod) weights[getType(dmgMod)][dmgMod] = (weights[getType(dmgMod)][dmgMod]||0) + weight;

    return weights;
  }, {other: {will: 1}, attributes:{strength:1, dexterity:1, vitality:25, special:1, insight:25}, skills:{brawl:1, channel: 1, clash:1, evasion: 1, alert: 1, athletic: 1, nature: 1, stealth: 1, allure: 1, etiquette: 1, intimidate: 1, perform: 1}, social:{tough:1, cool:1, beauty:1, cute:1, clever:1}});


  console.log(structuredClone({weights}));


  const compendiumSource = await fromUuid(actor._stats.compendiumSource);
  const stats = structuredClone({ 
    attributes: compendiumSource?.system?.attributes || actor.system.attributes,
    skills: compendiumSource?.system?.skills || actor.system.skills,
    social: compendiumSource?.system?.social || actor.system.social,
  });

  let skillsLeft = skillsRankData[rank].skills;
  let maxIterations = 100;
  while(skillsLeft > 0 && maxIterations-- > 0) {
    const skill = choseRandomWeighted(weights.skills);
    if(!skill) break;
    if(skillsRankData[rank].max === stats.skills[skill].value) {
      delete weights.skills[skill];
    } else {
      stats.skills[skill].value++; 
      skillsLeft--;
    }
  }

  let attributesLeft = skillsRankData[rank].attributes;
  maxIterations = 100;
  while(attributesLeft > 0 && maxIterations-- > 0) {
    const attribute = choseRandomWeighted(weights.attributes);
    if(!attribute) break;
    if(stats.attributes[attribute].max === stats.attributes[attribute].value) {
      delete weights.attributes[attribute];
    } else {
      stats.attributes[attribute].value++; 
      attributesLeft--;
    }
  }

  let socialLeft = skillsRankData[rank].social;
  maxIterations = 100;
  while(socialLeft > 0 && maxIterations-- > 0) {
    const socialKey = choseRandomWeighted(weights.social);
    if(!socialKey) break;
    if(stats.social[socialKey].max === stats.social[socialKey].value) {
      delete weights.social[socialKey];
    } else {
      stats.social[socialKey].value++; 
      socialLeft--;
    }
  }
  
  await actor.update({
    system: { 
      rank,
      ...stats
    },
  });

  console.log({actor, stats});

  await actor.update({
    system: {
      hp: { value: actor.system.hp.max },
      will: { value: actor.system.will.max }
    }
  });
}

const dragItem = (event) => {
  const data = { rank: event.target.name, type: 'PkmnFill' };
  event.dataTransfer.setData("text/plain", JSON.stringify(data));
}

const dragDrop = new DragDrop({
  dragSelector: ".fill-rank",
  callbacks: { dragstart: dragItem }
});

new Dialog({
title: 'PokéFill',
content: `
<div style="display:grid;grid-gap:3px;grid-template-columns:1fr 1fr 1fr">
<button class="fill-rank" name="starter">Starter</button>
<button class="fill-rank" name="beginner">Beginner</button>
<button class="fill-rank" name="amateur">Amateur</button>
<button class="fill-rank" name="ace">Ace</button>
<button class="fill-rank" name="pro">Pro</button>
<button class="fill-rank" name="master">Master</button>
<button class="fill-rank" name="champion">Champion</button>
</div>
<p><small><i>Drag n' derop the rank on the character sheet.</i></small></p>
`,
buttons:[],
render:(html) => {
  dragDrop.bind(html[0]);
  Hooks.on('dropActorSheetData', handleDrop);
},
close:() => {
  Hooks.off('dropActorSheetData', handleDrop);
}
}).render(true);
```

## [GM] PokéGen : Generates random Pokémons
**Macro type :** script
**Description :** Displays a dialog to generate random Pokémons
```js
const COUNT = 6; // Number of generated Pokémons
const LANG = 'en';

const typesFr = {
'bug': 'Bug', 'dark': 'Dark', 'dragon': 'Dragon',
'electric': 'Electric', 'fairy': 'Fairy', 'fighting': 'Fighting',
'fire': 'Fire', 'flying': 'Flying', 'ghost': 'Ghost',
'grass': 'Grass', 'ground': 'Ground', 'ice': 'Ice',
'normal': 'Normal', 'poison': 'Poison', 'psychic': 'Psychic',
'rock': 'Rock', 'steel': 'Steel', 'water': 'Water'
};
const types = (await (await fetch("https://pokeapi.co/api/v2/type/")).json()).results;

function shuffleArray(array) {
  let currentIndex = array.length;
  // While there remain elements to shuffle...
  while (currentIndex != 0) {
    // Pick a remaining element...
    let randomIndex = Math.floor(Math.random() * currentIndex);
    currentIndex--;
    // And swap it with the current element.
    [array[currentIndex], array[randomIndex]] = [
      array[randomIndex], array[currentIndex]];
  }
}

// select :
// - base / évolué
// - types

const fieldTypes = types.map(({name}) => typesFr[name] ? `<input type="checkbox" id="type${name}" name="${name}"><label class="type" for="type${name}"><img width="16" style="border:0" src="systems/pokerole/images/types/${name}.svg"> ${typesFr[name]}</label>` : '').join('');

async function chosePkmn(pokemons, data) {
  let chosenPkmn, pkmnData, pkmnSpecies, eligible;

  do {
    chosenPkmn = pokemons.pop();
    pkmnData = await (await fetch(chosenPkmn.pokemon.url)).json();
    pkmnSpecies = await (await fetch(pkmnData.species.url)).json();

    eligible = true;
    eligible &= (data.evolved || !pkmnSpecies.evolves_from_species);
    eligible &= !pkmnSpecies.is_legendary;
    eligible &= !pkmnSpecies.is_mythical;
  } while(!eligible && pokemons.length);
  
  pkmnForm = await (await fetch(pkmnData.forms[0].url)).json();
  return {chosenPkmn, pkmnData, pkmnForm, pkmnSpecies}
}

async function createPkmnElement({ pkmnData, pkmnSpecies, pkmnForm}) {
  const name = pkmnSpecies.names.find(({language}) => language.name == 'en').name;
  const formName = pkmnForm.form_names.find(({language}) => language.name == 'en')?.name;
  const nameFr = pkmnSpecies.names.find(({language}) => language.name == LANG).name;
  const formNameFr = pkmnForm.form_names.find(({language}) => language.name == LANG)?.name;
  const $el = document.createElement('div');

  const fullName = (formName ? `${name} (${formName})` : name);
  const fullNameFr = (formNameFr ? `${nameFr} (${formNameFr})` : nameFr);

  $el.className = "pkmn";
  $el.style = "display:flex;flex-direction:column;flex:1;align-items:center;justify-content:center";
  const pkmns = game.packs.get('pokerole.pokemon').index.filter((value) => value.name.startsWith(name) && value.type === 'pokemon');
  const pkmn = pkmns.find((value) => value.name === fullName);
  const pkmnRep = pkmns.find((value) => value.name === name);

  if(!pkmn && pkmnRep) $el.style.color = 'blue';
  const size = pkmn?.img ? 64 : pkmnData.sprites.front_default ? 96 : 64;
  $el.innerHTML = `<img height="${size}" width="${size}" style="border:0;object-fit:contain" src="${pkmn?.img || pkmnData.sprites.front_default || pkmnRep?.img}"> <b>${fullNameFr}</b> <i>${fullName}</i>`;

  if(pkmn || pkmnRep) {
    $el.draggable = true;
    $el.ondragstart = (e) => dragItem(e, (pkmn || pkmnRep).uuid);
    $el.style.cursor = "pointer";
  } else {
    $el.style.color = "red";
  }
  return $el;
}

const dragItem = (event, uuid) => {
  const data = { uuid, type: 'Actor' };
  event.dataTransfer.setData("text/plain", JSON.stringify(data));
}

async function doSubmit(event, displayAll) {
  const form = event.target;
  const formData = new FormDataExtended(form);
  const data = Object.fromEntries(formData);
  const hasType = types.some(({name}) => data[name]);
  
  const availablePokemonByTypes = await Promise.all(types.map(async ({name, url}) => {
    if(!data[name] && hasType) return [];
    else return (await (await fetch(url)).json()).pokemon;
  }));

  const flatAvailablePkmn = availablePokemonByTypes.reduce((flat, list) => flat.concat(list), []);

  shuffleArray(flatAvailablePkmn);
z
  const $pkmnResult = form.querySelector('.pkmn-result');
  $pkmnResult.innerHTML = '<div style="display:flex;align-items:center;justify-content:center"><i class="fas fa-spin fa-spinner"></i></div>'.repeat(COUNT);
  
  const promiseArray = [];
  for(let i = 0; i<COUNT;i++) {
    promiseArray.push(chosePkmn(flatAvailablePkmn, data).then(async (pkmn) => $pkmnResult.children[i].replaceWith(await createPkmnElement(pkmn))));
  }
}

const searchDialog = new Dialog({
  content: `<style>.pkmn[draggable]:hover{background:rgba(0,0,0,0.1)}.pkmn *{pointer-events:none}#pkmn-search{display:flex;flex-direction:column}#pkmn-search .labels{display:flex;grid-gap:3px;flex-wrap:wrap;align-items: stretch;margin-bottom:6px}#pkmn-search input{position:absolute;left: -9999px}#pkmn-search label{border-radius: 16px; cursor:pointer;display:flex;grid-gap:3px;border:2px solid transparent;padding:3px 6px}#pkmn-search input:checked+label{border-color: green}.pkmn-result>*{border: 1px solid}</style><form id="pkmn-search">
    <h4>Types</h4>
    <div class="labels">${fieldTypes}</div>
    <h4>Options</h4>
    <div class="labels"><input type="checkbox" name="evolved" id="evolved"><label for="evolved"><i class="fas fa-chevrons-up"></i> Include evolutions</label></div>
    <div class="buttons"><button><i class="fas fa-dice"></i> Generate ${COUNT} Pokémons</button>
    <div class="pkmn-result" style="margin-top:6px;height:${160*Math.ceil(COUNT/3)}px; display:grid;grid-gap:3px;grid-template-columns: 1fr 1fr 1fr;grid-auto-rows:1fr;align-items:stretch;justify-content: stretch;text-align:center">${`<div style="display:flex;align-items:center;justify-content:center"><i class="fas fa-circle-question" style="color:var(--color-text-dark-secondary);"></i></div>`.repeat(COUNT)}</div>
  </form>`,
  width: 450,
  buttons: {},
  render: (html) => {
    html.on('submit', 'form', doSubmit);
  },
}).render(true);
```

## Pokéballs
**Macro type :** chat
**Description :** Displays a chat message to throw Pokéballs, Greatballs and Ultraballs.
```js
<div style="height:0.5em"></div><img style="border: 0; margin:-10px -30px -10px -5px;z-index: 1; position:relative; pointer-events: none;" src="https://www.pokepedia.fr/images/0/07/Miniature_Pok%C3%A9_Ball_HOME.png"/> [[/sc 4]]{Pokéball} <img style="border: 0; margin:-10px -30px -10px -5px;z-index: 1; position:relative; pointer-events: none;" src="https://www.pokepedia.fr/images/2/23/Miniature_Super_Ball_HOME.png"/> [[/sc 6]]{Greatball} <img style="border: 0; margin:-10px -30px -10px -5px;z-index: 1; position:relative; pointer-events: none;" src="https://www.pokepedia.fr/images/a/a2/Miniature_Hyper_Ball_HOME.png"/> [[/sc 8]]{Ultraball}
<small style="display: block;margin-top: 6px">+1 succes if the Pokémon has half HP.
+1 success if the Pokémon has only 1 HP.
+1 success if the Pokémon is affected by a status effect (Paralysis, Poison...)</small>
```

## PokéTypes : Weaknesses and strengths table
**Macro type :** script
**Description :** Displays a dialog to quickly see what's strong or weak against types.
```js
const pokeapi = (url, base = 'https://pokeapi.co/api/v2/') => fetch(base+url).then((r) => r.json());
const translate = (array, lang = 'en') => array.find((item) => item.language.name === lang).name;

const apiTypes = (await pokeapi('type')).results.filter(({name}) => !['stellar', 'unknown'].includes(name));
const types = await Promise.all(apiTypes.map(({url}) => pokeapi(url, '')));

const checkType = (array, type) => array.find(({name}) => name === type);

let selectedType = null;

function selectType(type, html) {
  const $buttonSelect = html.querySelector(`.type-select[data-type=${type}]`);
  const typeData = types.find(({name}) => name === type);
  

  if(type !== selectedType) {
    if(selectedType) html.querySelector(`.type-select[data-type=${selectedType}]`).classList.remove('selected');
    $buttonSelect.classList.add('selected');
    selectedType = type;
    html.querySelector(`.selected-type`).innerHTML = `<img class="type" src="${typeData.sprites['generation-ix']['scarlet-violet'].name_icon}"/>`
  }

  // Afficher les forces, faiblesses, etc.
  const damageRelations = typeData.damage_relations;
  Object.entries(damageRelations).forEach(([key, relations]) => {
    displayDamageRelations(relations, html.querySelector(`.${key}`));
  });
  // Matchups
  const goodMatchups = types.filter(({name}) => (checkType(damageRelations.half_damage_from,name) || checkType(damageRelations.no_damage_from,name) || checkType(damageRelations.double_damage_to,name)) && !checkType(damageRelations.no_damage_to,name) && !checkType(damageRelations.double_damage_from,name) && !checkType(damageRelations.half_damage_to,name));

  const badMatchups = types.filter(({name}) => (checkType(damageRelations.half_damage_to,name) || checkType(damageRelations.no_damage_to,name) || checkType(damageRelations.double_damage_from,name)) && !checkType(damageRelations.double_damage_to,name) && !checkType(damageRelations.half_damage_from,name));
  displayDamageRelations(goodMatchups, html.querySelector('.good_matchups'));
  displayDamageRelations(badMatchups, html.querySelector('.bad_matchups'));
}

function displayDamageRelations(damageRelations, container) {
  container.innerHTML = damageRelations.map(({name}) => {
    const type = types.find((type) => name === type.name);
    return `<img class="type type-result" src="${type.sprites['generation-ix']['scarlet-violet'].name_icon}"/>`
  }).join('');
  if(!container.innerHTML) container.innerHTML = "-";
}

new Dialog({
 title: "PokéTypes",
 content: `<style>#pktp table td{width:33.333%;text-align:center}#pktp input{vertical-align:middle}#pktp .types{display:grid;grid-gap:3px;grid-template-columns:1fr 1fr 1fr 1fr}#pktp .type{border-radius: 24px;margin:auto;vertical-align: middle}#pktp .type-select{height: 22px;border:3px solid transparent;cursor:pointer}#pktp .type-select.selected{border-color:red} #pktp .type-select:hover{border-color:gray}#pktp .type-result{margin: 0 1px 1px;height:14px;border:0}#pktp .types-results{min-height: 200px}#pktp .types-results .positive b{color:green}#pktp .types-results .negative b{color:red}#pktp .types-results li{line-height:18px}#pktp .selected-type{margin:6px -8px;padding:6px 0;background:grey;text-align:center;position:sticky;top:-8px}#pktp .selected-type img{height:26px;}</style>
 <div id="pktp">
   <div class="types">
    ${types.map(({name, sprites}) => `<img class="type type-select" data-type="${name}" src="${sprites['generation-ix']['scarlet-violet'].name_icon}"/>`).join('')}
   </div>
   <div class="types-results">
     <div class="selected-type"></div>
     <table>
       <thead>
         <tr>
           <th>👍</th>
           <th>👎</th>
         </tr>
       </thead>
       <tbody>
         <tr>
           <td class="good_matchups"/>
           <td class="bad_matchups"/>
         </tr>
       </tbody>
     </table>
     <details><summary>See more</summary>
     <table>
       <thead>
         <tr>
           <th colspan="3">🛡️ Moves attacking this Pokémon type</th>
         </tr>
         <tr>
           <th>Strong</th>
           <th>Weak</th>
           <th>No effect</th>
         </tr>
       </thead>
       <tbody>
         <tr>
           <td class="double_damage_from"/>
           <td class="half_damage_from"/>
           <td class="no_damage_from"/>
         </tr>
       </tbody>
     </table>
     <table>
       <thead>
         <tr>
           <th colspan="3">⚔️ Moves of this type</th>
         </tr>
         <tr>
           <th>Strong</th>
           <th>Weak</th>
           <th>No effet</th>
         </tr>
       </thead>
       <tbody>
         <tr>
           <td class="double_damage_to"/>
           <td class="half_damage_to"/>
           <td class="no_damage_to"/>
         </tr>
       </tbody>
     </table>
     </details>
 </div>`,
 buttons: [],
 render: html => {
   // somthing like html.button.on('click', tatata)
   html.on('click', '.type-select', b => selectType(b.target.attributes['data-type'].value, html[0]));
   selectType('normal', html[0])
 },
}).render(true);
```
