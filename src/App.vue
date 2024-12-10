<template>
  <div id="app">
    <h1>Business Intelligence Dashboard</h1>

    <!-- Bottone per mostrare/nascondere il menu -->
    <button @click="toggleFieldSelector">
      {{ showFieldSelector ? "Nascondi Opzioni" : "Mostra Opzioni" }}
    </button>
    <!-- Bottone per esportare il file -->
    <button @click="exportToExcel">Esporta in Excel</button>

    <!-- Menu per selezionare campi aggiuntivi -->
    <div v-if="showFieldSelector" class="field-selector">
      <label>Seleziona campi aggiuntivi da visualizzare:</label>
      <div v-for="field in availableFields" :key="field.key" class="checkbox">
        <input
          type="checkbox"
          :id="field.key"
          :value="field.key"
          v-model="selectedFields"
        />
        <label :for="field.key">{{ field.label }}</label>
      </div>
    </div>

    <!-- Visualizzazione Raggruppata -->
    <div v-if="viewMode === 'table' && groupedData.length">
      <h2>Dati Raggruppati per Business Actor</h2>
      <div v-for="(group, index) in groupedData" :key="index" class="group">
        <h3>Responsabile: {{ group.businessActor }}</h3>
        <table border="1">
          <thead>
            <tr>
              <th>Tavolo</th>
              <th>Prodotto</th>
              <th>Prezzo Unitario</th>
              <th>Quantità</th>
              <th v-if="isFieldSelected('totalPrice')">Prezzo Complessivo</th>
              <th v-if="isFieldSelected('productCategory')">Categoria prodotto</th>
              <th v-if="isFieldSelected('businessActorName')">Nome responsabile</th>
              <th v-if="isFieldSelected('tableNumber')">Numero tavolo</th>
              <th v-if="isFieldSelected('docNumber')">Numero documento</th>
              <th v-if="isFieldSelected('guestCount')">Quantità ospiti</th>
              <th v-if="isFieldSelected('stayDuration')">Durata permanenza</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="(item, itemIndex) in group.items" :key="`${item.tableName}-${itemIndex}`">
              <tr>
                <td>{{ item.tableName }}</td>
                <td>{{ item.product }}</td>
                <td>{{ item.computedUnitaryPrice.toFixed(2) }} €</td>
                <td>{{ item.quantity }}</td>
                <td v-if="isFieldSelected('totalPrice')">{{ (item.computedUnitaryPrice * item.quantity).toFixed(2) }} €</td>
                <td v-if="isFieldSelected('productCategory')">{{ item.category }}</td>
                <td v-if="isFieldSelected('businessActorName')">{{ group.businessActor }}</td>
                <td v-if="isFieldSelected('tableNumber')">{{ item.tableNumber }}</td>
                <td v-if="isFieldSelected('docNumber')">{{ item.docNumber }}</td>
                <td v-if="isFieldSelected('guestCount')">{{ item.guestCount }}</td>
                <td v-if="isFieldSelected('stayDuration')">{{ item.stayDuration }} minuti</td>
              </tr>
              <tr
                v-if="itemIndex === group.items.length - 1 || item.tableName !== group.items[itemIndex + 1]?.tableName"
              >
                <td colspan="3"><strong>Totale Tavolo: {{ calculateTableTotal(group.items, item.tableName).toFixed(2) }} €</strong></td>
                <td colspan="3"><strong>Data: {{ item.date }}</strong></td>
              </tr>
            </template>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import * as XLSX from "xlsx";

export default {
  data() {
    return {
      groupedData: [],
      viewMode: "table",
      selectedFields: [], // Campi aggiuntivi selezionati
      showFieldSelector: false, // Controlla la visibilità del menu a tendina
      availableFields: [
        { key: "totalPrice", label: "Prezzo complessivo" },
        { key: "productCategory", label: "Categoria prodotto" },
        { key: "businessActorName", label: "Nome responsabile" },
        { key: "tableNumber", label: "Numero tavolo" },
        { key: "docNumber", label: "Numero documento" },
        { key: "guestCount", label: "Quantità ospiti" },
        { key: "stayDuration", label: "Durata permanenza" },
      ],
    };
  },
  methods: {
    toggleFieldSelector() {
      this.showFieldSelector = !this.showFieldSelector;
    },
    async loadClosedPayments() {
      try {
        const response = await fetch("/closedpayments.json"); // Assicurati che il file sia nella cartella "public"
        const data = await response.json();
        this.groupedData = this.groupOrdersByBusinessActor(data);
      } catch (error) {
        console.error("Errore nel caricamento del file JSON:", error);
      }
    },
    groupOrdersByBusinessActor(data) {
      const grouped = [];

      data.forEach((session) => {
        const businessActor = session.value?.businessActor?.name || "Sconosciuto";
        const referenceDate = session.value?.referenceDate || "Data non disponibile"; // Prende la data dal session

        const items = session.value?.printedOrderItems?.map((item) => ({
          product: item.orderItemName || "Prodotto non disponibile",
          computedUnitaryPrice: item.computedUnitaryPrice || 0,
          quantity: item.quantity || 1,
          category: item.product?.category?.name || "Categoria non disponibile",
          tableName: session.value?.table?.name || "Tavolo non disponibile",
          tableNumber: session.value?.table?.id || "N/A",
          docNumber: session.value?.billNumber || "N/A",
          guestCount: session.value?.table?.numberOfGuests || 0,
          stayDuration: Math.round((session.value?.table?.stayingDuration || 0) / 60), // Converti secondi in minuti
          date: referenceDate, // Associare la data corretta
        })) || [];

        const existingGroup = grouped.find((group) => group.businessActor === businessActor);
        if (existingGroup) {
          existingGroup.items.push(...items);
        } else {
          grouped.push({
            businessActor,
            items,
          });
        }
      });

      return grouped;
    },
    calculateTableTotal(items, tableName) {
      return items
        .filter((item) => item.tableName === tableName)
        .reduce((total, item) => total + item.computedUnitaryPrice * item.quantity, 0);
    },
    isFieldSelected(field) {
      return this.selectedFields.includes(field);
    },
    exportToExcel() {
      const exportData = [];
      this.groupedData.forEach((group) => {
        group.items.forEach((item) => {
          exportData.push({
            "Business Actor": group.businessActor,
            Tavolo: item.tableName,
            Prodotto: item.product,
            "Prezzo Unitario": item.computedUnitaryPrice.toFixed(2),
            Quantità: item.quantity,
            "Prezzo Complessivo": (item.computedUnitaryPrice * item.quantity).toFixed(2),
            Categoria: item.category,
            "Numero Tavolo": item.tableNumber,
            "Numero Documento": item.docNumber,
            "Quantità Ospiti": item.guestCount,
            "Durata Permanenza (minuti)": item.stayDuration,
            Data: item.date,
          });
        });
      });

      const worksheet = XLSX.utils.json_to_sheet(exportData);
      const workbook = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(workbook, worksheet, "Dati Raggruppati");
      XLSX.writeFile(workbook, "Dati_Raggruppati.xlsx");
    },
  },
  mounted() {
    this.loadClosedPayments();
  },
};
</script>

<style scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  margin-top: 20px;
}
button {
  margin-bottom: 20px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
.field-selector {
  margin-bottom: 20px;
  border: 1px solid #ccc;
  padding: 10px;
  background-color: #f9f9f9;
}
.checkbox {
  margin: 5px 0;
}
table {
  margin-top: 10px;
  width: 100%;
  border-collapse: collapse;
}
th,
td {
  padding: 8px;
  text-align: center;
}
.group {
  margin-bottom: 20px;
}
</style>
