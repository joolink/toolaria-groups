<template>
  <div class="p-6 max-w-xl mx-auto">
    <h1 class="text-2xl font-bold mb-4">Gruppen-Tool</h1>

    <div class="mb-4">
      <label class="block font-semibold mb-1">Liste der Namen (je Name pro Zeile):</label>
      <textarea v-model="namesInput" rows="5" class="border p-2 rounded w-full mb-2" placeholder="Namen..."></textarea>
    </div>

    <div class="mb-4 flex gap-4">
      <div>
        <label class="block font-semibold mb-1">Gruppengröße:</label>
        <input v-model.number="groupSize" type="number" min="1" class="border p-2 rounded w-24" />
      </div>
      <div>
        <label class="block font-semibold mb-1">Max. Gruppenanzahl:</label>
        <input v-model.number="maxGroups" type="number" min="1" class="border p-2 rounded w-24" />
      </div>
    </div>

    <div class="mb-4">
      <label class="block font-semibold mb-1">Rollen (je Rolle pro Zeile):</label>
      <textarea v-model="rolesInput" rows="3" class="border p-2 rounded w-full mb-2" placeholder="Rollen..."></textarea>
    </div>

    <div class="card p-4 mb-4">
      <label class="block font-semibold mb-1">Kombinationen aus der Historie verbieten:
        <input type="checkbox" v-model="forbidHistoryCombos" class="ml-2" />
      </label>
    </div>

    <button @click="makeGroups" class="bg-blue-500 text-white px-4 py-2 rounded mb-4">Gruppen erstellen</button>

    <button @click="generatePdf" class="bg-green-600 text-white px-4 py-2 rounded mb-4">PDF generieren (aktuelle Gruppen)</button>
    <div v-if="groupResults.length" class="mb-6">
      <h2 class="text-xl font-bold mb-2">Erstellte Gruppen</h2>
      <div v-for="(group, i) in groupResults" :key="i" class="mb-4 card p-3">
        <div class="font-semibold mb-2">Gruppe {{ i + 1 }}</div>
        <table class="w-full text-left border-collapse">
          <thead>
            <tr>
              <th class="py-1 px-2 border-b">Name</th>
              <th class="py-1 px-2 border-b">Rolle</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(member, j) in group" :key="j">
              <td class="py-1 px-2 border-b">{{ member.name }}</td>
              <td class="py-1 px-2 border-b">{{ member.role || '-' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <button @click="exportHistory" class="bg-gray-700 text-white px-4 py-2 rounded mb-4">Historie exportieren (JSON)</button>
    <div class="flex items-center gap-2 mb-2">
      <button @click="toggleHistory" class="bg-blue-500 text-white px-3 py-1 rounded">
        {{ showHistory ? 'Historie zuklappen' : 'Historie anzeigen' }}
      </button>
      <button @click="clearHistory" class="bg-red-600 text-white px-3 py-1 rounded">
        Historie löschen
      </button>
      <label class="bg-gray-700 text-white px-3 py-1 rounded cursor-pointer">
        Historie importieren
        <input type="file" accept="application/json" @change="importHistory" style="display:none" />
      </label>
    </div>
    <div v-if="showHistory && history.length" class="mb-6">
      <h2 class="text-xl font-bold mb-2">Historie</h2>
      <div v-for="h in history" :key="h.id" class="mb-4 border rounded p-3 bg-gray-50">
        <div class="font-semibold mb-1">ID: {{ h.id }}</div>
        <div class="text-xs mb-1">Zeit: {{ new Date(h.timestamp).toLocaleString() }}</div>
        <button @click="generateHistoryPdf(h)" class="bg-green-600 text-white px-2 py-1 rounded mb-2">PDF für diese Gruppe</button>
        <div v-for="(group, i) in h.groups" :key="i" class="mb-2">
          <div class="font-semibold">Gruppe {{ i + 1 }}</div>
          <ul>
            <li v-for="(member, j) in group" :key="j">
              {{ member.name }}<span v-if="member.role"> ({{ member.role }})</span>
            </li>
          </ul>
        </div>
      </div>
    </div>

    <div class="mb-4">
      <label class="block font-semibold mb-1">Historie importieren (JSON-Datei):</label>
      <input type="file" @change="importHistory" class="border p-2 rounded w-full" />
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import { jsPDF } from "jspdf";

// Inputs
const namesInput = ref("");
const groupSize = ref(3);
const maxGroups = ref(0); // 0 means not set
const rolesInput = ref("");
const forbidHistoryCombos = ref(false);

// Results
const groupResults = ref([]);
const history = ref(JSON.parse(localStorage.getItem("groupHistory") || "[]"));
const showHistory = ref(true);

function makeGroups() {
  let names = namesInput.value.split(/\r?\n/).map(n => n.trim()).filter(n => n);
  let roles = rolesInput.value.split(/\r?\n/).map(r => r.trim()).filter(r => r);
  let groups = [];

  // Shuffle names for randomness
  names = names.sort(() => Math.random() - 0.5);

  let numGroups = maxGroups.value > 0 ? maxGroups.value : Math.ceil(names.length / groupSize.value);
  let size = groupSize.value > 0 ? groupSize.value : Math.ceil(names.length / numGroups);

  for (let i = 0; i < numGroups; i++) groups.push([]);
  names.forEach((name, idx) => {
    groups[idx % numGroups].push({ name });
  });

  // Assign roles
  groups.forEach(group => {
    group.forEach((member, idx) => {
      if (roles[idx]) member.role = roles[idx];
    });
  });

  // Kombinationsprüfung
  if (forbidHistoryCombos.value) {
    // Kombiniere Gruppengröße, Namen und Rollen zu einem eindeutigen String pro Gruppe
    const currentCombos = groups.map(g => {
      const members = g.map(m => `${m.name}:${m.role || ''}`).sort().join('|');
      return `size:${g.length}|${members}`;
    });
    const historyCombos = history.value.flatMap(h => h.groups.map(g => {
      const members = g.map(m => `${m.name}:${m.role || ''}`).sort().join('|');
      return `size:${g.length}|${members}`;
    }));
    const hasForbidden = currentCombos.some(combo => historyCombos.includes(combo));
    if (hasForbidden) {
      alert('Keine neuen Gruppenkombinationen möglich! Bitte Option deaktivieren, um weitere Gruppen zu erzeugen.');
      return;
    }
  }

  groupResults.value = groups;

  // Save to history
  const entry = {
    id: `${Date.now()}-${Math.random().toString(36).slice(2,8)}`,
    timestamp: Date.now(),
    groups: JSON.parse(JSON.stringify(groups)),
    names: [...names],
    roles: [...roles],
    groupSize: groupSize.value,
    maxGroups: maxGroups.value
  };
  history.value.unshift(entry);
  localStorage.setItem("groupHistory", JSON.stringify(history.value));
}

function generatePdf() {
  const doc = new jsPDF();
  doc.text("Erstellte Gruppen:", 10, 10);
  let y = 20;
  groupResults.value.forEach((group, i) => {
    doc.text(`Gruppe ${i + 1}:`, 10, y);
    y += 8;
    // Tabellenkopf
    doc.text('Name', 14, y);
    doc.text('Rolle', 64, y);
    y += 6;
    // Tabelleninhalt
    group.forEach(member => {
      doc.text(member.name, 14, y);
      doc.text(member.role ? member.role : '-', 64, y);
      y += 6;
    });
    y += 6;
  });
  doc.save("gruppen.pdf");
}

function exportHistory() {
  const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(history.value, null, 2));
  const a = document.createElement('a');
  a.setAttribute('href', dataStr);
  a.setAttribute('download', 'gruppen-historie.json');
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
}

function generateHistoryPdf(entry) {
  const doc = new jsPDF();
  doc.text("Gruppen Historie:", 10, 10);
  doc.text(`ID: ${entry.id}`, 10, 18);
  doc.text(`Zeit: ${new Date(entry.timestamp).toLocaleString()}`, 10, 26);
  let y = 36;
  entry.groups.forEach((group, i) => {
    doc.text(`Gruppe ${i + 1}:`, 10, y);
    y += 8;
    doc.text('Name', 14, y);
    doc.text('Rolle', 64, y);
    y += 6;
    group.forEach(member => {
      doc.text(member.name, 14, y);
      doc.text(member.role ? member.role : '-', 64, y);
      y += 6;
    });
    y += 6;
  });
  doc.save(`gruppen_${entry.id}.pdf`);
}

function toggleHistory() {
  showHistory.value = !showHistory.value;
}

function clearHistory() {
  if (confirm('Möchtest du wirklich die gesamte Historie löschen?')) {
    history.value = [];
    localStorage.removeItem('groupHistory');
  }
}

function importHistory(event) {
  const file = event.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const imported = JSON.parse(e.target.result);
      if (Array.isArray(imported)) {
        history.value = imported;
        localStorage.setItem('groupHistory', JSON.stringify(imported));
        alert('Historie erfolgreich importiert!');
      } else {
        alert('Ungültiges Format. Die Datei muss ein Array von Historie-Einträgen enthalten.');
      }
    } catch {
      alert('Fehler beim Einlesen der Datei.');
    }
  };
  reader.readAsText(file);
}
</script>

<style>
/* Modern CSS for Gruppen-Tool */
body {
  font-family: 'Inter', 'Segoe UI', Arial, sans-serif;
  background: #f6f8fa;
  margin: 0;
}

.p-6 {
  background: #fff;
  border-radius: 18px;
  box-shadow: 0 4px 24px rgba(0,0,0,0.08);
  margin-top: 32px;
  padding: 32px;
}

h1, h2 {
  color: #1a202c;
}

label {
  color: #374151;
}

input, textarea {
  border: 1px solid #d1d5db;
  border-radius: 8px;
  padding: 10px;
  font-size: 1rem;
  background: #f9fafb;
  transition: border-color 0.2s;
}
input:focus, textarea:focus {
  border-color: #2563eb;
  outline: none;
}

button {
  transition: background 0.2s, box-shadow 0.2s;
  box-shadow: 0 2px 8px rgba(37,99,235,0.08);
}
.bg-blue-500 {
  background: linear-gradient(90deg, #2563eb 0%, #1e40af 100%);
}
.bg-blue-500:hover {
  background: linear-gradient(90deg, #1e40af 0%, #2563eb 100%);
}
.bg-green-600 {
  background: linear-gradient(90deg, #059669 0%, #065f46 100%);
}
.bg-green-600:hover {
  background: linear-gradient(90deg, #065f46 0%, #059669 100%);
}
.bg-gray-700 {
  background: #374151;
}
.bg-gray-700:hover {
  background: #111827;
}
.text-white {
  color: #fff;
}
.rounded {
  border-radius: 8px;
}

.card, .border {
  background: #f9fafb;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.04);
  border: 1px solid #e5e7eb;
}

ul {
  padding-left: 1.2em;
}

.mb-4.flex {
  display: block;
}
.mb-4.flex > div {
  margin-bottom: 16px;
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 8px;
}
th, td {
  padding: 8px 12px;
  border: 1px solid #e5e7eb;
}
th {
  background: #f1f5f9;
  color: #111827;
  text-align: left;
}

@media (max-width: 600px) {
  .p-6 {
    padding: 12px;
    margin-top: 8px;
  }
  input, textarea {
    font-size: 0.95rem;
    padding: 8px;
  }
}
</style>
