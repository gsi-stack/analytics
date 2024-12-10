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
              <th v-if="isFieldSelected('unitaryPrice')">Prezzo Unitario</th>
              <th>Prezzo Complessivo</th>
              <th>Prezzo Scontato</th>
              <th>VAT (%)</th>
              <th>Operatore</th>
              <th>Quantità</th>
              <th v-if="isFieldSelected('productCategory')">Categoria prodotto</th>
              <th v-if="isFieldSelected('productCode')">Codice Prodotto</th>
              <th v-if="isFieldSelected('accessNotes')">Note Tavolo</th>
              <th v-if="isFieldSelected('arrivalTime')">Ora di Arrivo</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="(item, itemIndex) in group.items" :key="`${item.tableName}-${itemIndex}`">
              <tr>
                <td>{{ item.tableName }}</td>
                <td>{{ item.product }}</td>
                <td v-if="isFieldSelected('unitaryPrice')">{{ item.computedUnitaryPrice.toFixed(2) }} €</td>
                <td>{{ (item.computedUnitaryPrice * item.quantity).toFixed(2) }} €</td>
                <td>{{ item.discountedPrice.toFixed(2) }} €</td>
                <td>{{ item.vat }}</td>
                <td>{{ item.businessMember }}</td>
                <td>{{ item.quantity }}</td>
                <td v-if="isFieldSelected('productCategory')">{{ item.category }}</td>
                <td v-if="isFieldSelected('productCode')">{{ item.productCode }}</td>
                <td v-if="isFieldSelected('accessNotes')">{{ item.accessNotes }}</td>
                <td v-if="isFieldSelected('arrivalTime')">{{ item.formattedArrivalTime }}</td>
              </tr>
              <!-- RIGA AGGIUNTIVA: Totale del Tavolo e Data -->
              <tr v-if="itemIndex === group.items.length - 1 || item.tableName !== group.items[itemIndex + 1]?.tableName">
                <td colspan="4"><strong>Totale Tavolo: {{ calculateTableTotal(group.items, item.tableName).toFixed(2) }} €</strong></td>
                <td colspan="4"><strong>Data: {{ item.date }}</strong></td>
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
      businessMembers: {}, // Mappa per ID -> Nickname
      viewMode: "table",
      selectedFields: [], // Campi aggiuntivi selezionati
      showFieldSelector: false, // Controlla la visibilità del menu a tendina
      availableFields: [
        { key: "unitaryPrice", label: "Prezzo unitario" },
        { key: "productCategory", label: "Categoria prodotto" },
        { key: "productCode", label: "Codice prodotto" },
        { key: "accessNotes", label: "Note Tavolo" },
        { key: "arrivalTime", label: "Ora di arrivo" },
      ],
    };
  },
  methods: {
    toggleFieldSelector() {
      this.showFieldSelector = !this.showFieldSelector;
    },
    async loadBusinessMembers() {
      try {
        const response = await fetch("/businessMembers.json");
        const data = await response.json();
        // Creiamo un mapping ID -> nickname
        this.businessMembers = data.reduce((map, member) => {
          map[member.id] = member.value.nickname || "Sconosciuto";
          return map;
        }, {});
      } catch (error) {
        console.error("Errore nel caricamento dei dati dei business members:", error);
      }
    },
    async loadClosedPayments() {
      try {
        const response = await fetch("/closedpayments.json");
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
        const referenceDate = session.value?.referenceDate || "Data non disponibile";

        const items = session.value?.printedOrderItems?.map((item) => ({
          product: item.orderItemName || "Prodotto non disponibile",
          computedUnitaryPrice: item.computedUnitaryPrice || 0,
          discountedPrice: item.finalPriceWithSessionDiscounts || 0,
          vat: item.vatRecordCategory?.rate || "N/A",
          businessMember:
            this.businessMembers[session.value?.businessMemberId] || "Operatore sconosciuto",
          quantity: item.quantity || 1,
          category: item.product?.category?.name || "Categoria non disponibile",
          tableName: session.value?.table?.name || "Tavolo non disponibile",
          productCode: item.product?.productId || "N/A",
          accessNotes: session.value?.table?.accessNotes || "N/A",
          formattedArrivalTime: new Date(session.value?.originalOrderLogCreatedDate).toLocaleTimeString("it-IT"),
          date: referenceDate,
        }));

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
            Tavolo: item.tableName,
            Prodotto: item.product,
            "Prezzo Unitario": item.computedUnitaryPrice.toFixed(2),
            "Prezzo Complessivo": (item.computedUnitaryPrice * item.quantity).toFixed(2),
            "Prezzo Scontato": item.discountedPrice.toFixed(2),
            VAT: item.vat,
            Operatore: item.businessMember,
            Quantità: item.quantity,
            "Codice Prodotto": item.productCode,
            "Note Tavolo": item.accessNotes,
            "Ora di Arrivo": item.formattedArrivalTime,
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
  async mounted() {
    await this.loadBusinessMembers(); // Carichiamo i nickname prima di caricare i pagamenti
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
