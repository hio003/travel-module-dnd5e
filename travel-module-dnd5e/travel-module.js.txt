// travel-module.js

/**
 * Travel Rules Module for Foundry VTT (D&D 5e)
 * Inspired by The One Ring 2e Journey mechanics
 * Adds travel roles, fatigue tracking, and random events
 */

Hooks.once("ready", async function () {
  if (!game.modules.get("dnd5e")) return;

  console.log("[Travel Module] Ready");

  // Register Travel Macros
  game.travels = {
    assignRoles: assignTravelRoles,
    rollTravelEvents: rollTravelEvent,
    fatigueCheck: fatigueCheckAll,
  };

  // Register rollable tables
  await createTravelEventTable();
});

async function assignTravelRoles() {
  const players = game.users.players.filter(u => u.character);
  const roles = ["Guía", "Explorador", "Cazador", "Centinela", "Portador"];

  let content = `<form>`;
  for (let i = 0; i < roles.length; i++) {
    content += `<div><label>${roles[i]}</label><select name="${roles[i]}">`;
    players.forEach(p => {
      content += `<option value="${p.character.id}">${p.name}</option>`;
    });
    content += `</select></div>`;
  }
  content += `</form>`;

  new Dialog({
    title: "Asignar Roles de Viaje",
    content,
    buttons: {
      ok: {
        label: "Asignar",
        callback: async (html) => {
          roles.forEach(role => {
            const charId = html.find(`[name='${role}']`).val();
            const actor = game.actors.get(charId);
            actor.setFlag("travel-module", "role", role);
          });
          ui.notifications.info("Roles de viaje asignados.");
        },
      },
    },
  }).render(true);
}

async function fatigueCheckAll() {
  for (let actor of game.actors.contents) {
    if (!actor.hasPlayerOwner) continue;
    const conMod = actor.data.data.abilities.con.mod;
    const roll = await new Roll("1d20 + @mod", { mod: conMod }).roll({ async: true });
    roll.toMessage({
      speaker: ChatMessage.getSpeaker({ actor }),
      flavor: "Chequeo de Fatiga",
    });
    if (roll.total < 12) {
      let level = actor.data.data.attributes.exhaustion ?? 0;
      actor.update({ "data.attributes.exhaustion": level + 1 });
      ChatMessage.create({ content: `${actor.name} gana un nivel de agotamiento.` });
    }
  }
}

async function createTravelEventTable() {
  if (game.tables.getName("Eventos de Viaje")) return;
  const results = [
    "Lluvia persistente inunda caminos. Penaliza movimiento.",
    "Un grupo de viajeros necesita ayuda o guía.",
    "Ruinas antiguas aparecen en el camino. ¿Explorar o evitar?",
    "Rastro de una bestia gigante o dragón.",
    "Extraña enfermedad afecta a uno de los personajes.",
    "Encuentro con bandidos o patrulla de orcos.",
    "Rumores o sueños inquietantes durante el descanso.",
    "El grupo se pierde. Repetir el día.",
    "Avance fácil. Recuperan nivel de agotamiento.",
    "Se descubre un atajo. El viaje se acorta.",
    "Visión sobrenatural o augurio.",
    "Encuentro con un sabio, druida o espíritu del bosque."
  ].map((text, i) => ({ type: 1, text, weight: 1, range: [i + 1, i + 1] }));

  await RollTable.create({
    name: "Eventos de Viaje",
    results,
    replacement: true,
    displayRoll: true,
    formula: "1d12",
  });
}

async function rollTravelEvent() {
  const table = game.tables.getName("Eventos de Viaje");
  if (!table) return ui.notifications.warn("La tabla 'Eventos de Viaje' no existe.");
  await table.draw();
}

// Macro helpers
// Use game.travels.assignRoles(), game.travels.rollTravelEvents(), game.travels.fatigueCheck()
