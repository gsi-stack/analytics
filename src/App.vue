<template>
  <div id="app">
    <h1>Business Intelligence Dashboard</h1>

    <!-- Modulo di Login -->
    <div v-if="!isAuthenticated" class="login-form">
      <h2>Accedi</h2>
      <input type="email" v-model="inputEmail" placeholder="Email" />
      <input type="password" v-model="inputPassword" placeholder="Password" />
      <button @click="login">Accedi</button>
      <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    </div>

    <!-- Dashboard principale dopo login -->
    <div v-else>
      <button @click="logout">Logout</button>
      <input type="file" @change="handleFileUpload" accept=".xlsx, .xls" />

      <div v-if="groupedData.length" class="view-selector">
        <label for="viewMode">Seleziona visualizzazione:</label>
        <select id="viewMode" v-model="viewMode">
          <option value="table">Visualizzazione Raggruppata</option>
          <option value="charts">Grafici</option>
        </select>
        <button @click="exportToExcel">Esporta in Excel</button>
      </div>

      <!-- Visualizzazione Raggruppata -->
      <div v-if="viewMode === 'table' && groupedData.length">
        <h2>Dati Raggruppati</h2>
        <div v-for="(group, index) in groupedData" :key="index" class="group">
          <h3>Gruppo: {{ group.identifier }}</h3>
          <table border="1">
            <thead>
              <tr>
                <th>Data doc.</th>
                <th>Numero doc.</th>
                <th>Tavolo</th>
                <th>Piatto</th>
                <th>Descrizione</th>
                <th>Quantità</th>
                <th>Importo</th>
                <th>Imp. scontato</th>
                <th>Nominativo</th>
                <th>Ora Arrivo</th>
                <th>Ora Arrivo</th>
                <th>nominativo offerto</th>
                <th>Gruppo</th>
                <th>Nome Gruppo</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, rowIndex) in group.rows" :key="rowIndex">
                <td>{{ formatDate(row["Data doc."]) }}</td>
                <td>{{ row["Numero doc."] }}</td>
                <td>{{ row.Tavolo }}</td>
                <td>{{ row.Piatto }}</td>
                <td>{{ row.Descrizione }}</td>
                <td>{{ row.Quantità }}</td>
                <td>{{ row.Importo }}</td>
                <td>{{ row["Imp. scontato"] }}</td>
                <td>{{ row.Nominativo }}</td>
                <td>{{ formatTime(row["Ora Arrivo"]) }}</td>
                <td>{{ formatTime(row["Ora Arrivo"]) }}</td>
                <td>{{ row["nominativo offerto"] || "" }}</td>
                <td>{{ group.identifier.split(" ")[0] }}</td>
                <td>{{ group.identifier.split(" ")[1] }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as XLSX from 'xlsx';

export default {
  data() {
    return {
      // Variabili di autenticazione predefinite
      correctEmail: "test@example.com",
      correctPassword: "password123",
      inputEmail: "",
      inputPassword: "",
      isAuthenticated: false,
      errorMessage: "",

      // Variabili dashboard
      groupedData: [],
      viewMode: 'table',
    };
  },
  methods: {
    // Funzione di autenticazio
    login() {
      if (this.inputEmail === this.correctEmail && this.inputPassword === this.correctPassword) {
        this.isAuthenticated = true;
        this.errorMessage = "";
      } else {
        this.errorMessage = "Email o password errate. Riprova.";
      }
    },
    logout() {
      this.isAuthenticated = false;
      this.inputEmail = "";
      this.inputPassword = "";
    },

    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const data = new Uint8Array(e.target.result);
          const workbook = XLSX.read(data, { type: 'array', raw: true });
          const sheetName = workbook.SheetNames[0];
          const worksheet = workbook.Sheets[sheetName];
          const jsonData = XLSX.utils.sheet_to_json(worksheet, { defval: null });
          this.groupedData = this.groupRows(jsonData);
        };
        reader.readAsArrayBuffer(file);
      }
    },

    groupRows(data) {
      const grouped = [];
      let currentGroup = null;

      data.forEach(row => {
        if (Object.values(row).every(value => value === null || value === "")) return;

        if (row["Data doc."] && row["Numero doc."] && !row["Descrizione"]) {
          if (currentGroup) grouped.push(currentGroup);
          currentGroup = {
            identifier: `${this.formatDate(row["Data doc."])} ${row["Numero doc."]}`,
            rows: []
          };
        } else if (currentGroup) {
          currentGroup.rows.push(row);
        }
      });

      if (currentGroup) grouped.push(currentGroup);
      return grouped;
    },

    exportToExcel() {
      const exportData = [];
      this.groupedData.forEach(group => {
        group.rows.forEach(row => {
          exportData.push({
            "Data doc.": this.formatDate(row["Data doc."]),
            "Numero doc.": row["Numero doc."],
            "Tavolo": row["Tavolo"],
            "Piatto": row["Piatto"],
            "Descrizione": row["Descrizione"],
            "Quantità": row["Quantità"],
            "Importo": row["Importo"],
            "Imp. scontato": row["Imp. scontato"],
            "Nominativo": row["Nominativo"],
            "Ora Arrivo": this.formatTime(row["Ora Arrivo"]),
            "Ora Arrivo.1": this.formatTime(row["Ora Arrivo"]),
            "nominativo offerto": row["nominativo offerto"] || "",
            "Gruppo": group.identifier.split(" ")[0],
            "Nome Gruppo": group.identifier.split(" ")[1]
          });
        });
      });

      const worksheet = XLSX.utils.json_to_sheet(exportData);
      const workbook = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(workbook, worksheet, "Dati Raggruppati");
      XLSX.writeFile(workbook, "Dati_Raggruppati_Stilizzati.xlsx");
    },

    formatDate(serial) {
      if (typeof serial === "number") {
        const date = new Date(Math.round((serial - 25569) * 86400 * 1000));
        const day = String(date.getUTCDate()).padStart(2, '0');
        const month = String(date.getUTCMonth() + 1).padStart(2, '0');
        const year = date.getUTCFullYear();
        return `${day}/${month}/${year}`;
      }
      return serial;
    },
    formatTime(serial) {
      if (typeof serial === "number") {
        const totalSeconds = Math.floor(serial * 86400);
        const hours = String(Math.floor(totalSeconds / 3600)).padStart(2, '0');
        const minutes = String(Math.floor((totalSeconds % 3600) / 60)).padStart(2, '0');
        const seconds = String(totalSeconds % 60).padStart(2, '0');
        return `${hours}:${minutes}:${seconds}`;
      }
      return serial;
    }
  }
};
</script>

<style scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  margin-top: 20px;
}
input[type="file"] {
  margin: 20px;
}
.view-selector {
  margin: 20px 0;
}
button {
  margin-top: 20px;
}
table {
  margin-top: 10px;
  width: 100%;
  border-collapse: collapse;
}
th, td {
  padding: 8px;
  text-align: center;
}
.group {
  margin-bottom: 20px;
}
.error {
  color: red;
}
</style>
