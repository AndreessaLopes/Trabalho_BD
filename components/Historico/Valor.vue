<template>
  <div>
    <v-app-bar fixed color="#041022" dark>
      <v-app-bar-nav-icon @click="toggleDrawer"></v-app-bar-nav-icon>
      <v-app-bar-title class="mr-6">Histórico de Preços de Ativos</v-app-bar-title>
      <v-icon @click="print" class="ml-auto">mdi-printer</v-icon>
    </v-app-bar>

    <v-navigation-drawer v-model="drawer" temporary clipped app>
      <v-list dense>
        <v-list-item v-for="item in items" :key="item.text" :to="item.link">
          <v-list-item-icon>
            <v-icon>{{ item.icon }}</v-icon>
          </v-list-item-icon>
          <v-list-item-content>
            <v-list-item-title>{{ item.text }}</v-list-item-title>
          </v-list-item-content>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>

    <v-main>
      <v-container>
        <v-data-table
          v-if="!loading && prices.length > 0"
          :headers="headers"
          :items="prices"
          item-key="id"
          class="elevation-1"
        >
          <template v-slot:item="{ item }">
            <tr class="bordered-row">
              <td v-for="(header, index) in headers" :key="index" class="cell-center">
                <span :class="header.class">{{ formatValue(item[header.value], header.type) }}</span>
              </td>
            </tr>
          </template>
        </v-data-table>
        <div v-else-if="loading" class="loading-message">Carregando dados...</div>
        <div v-else class="loading-message">Nenhum dado disponível.</div>
      </v-container>
    </v-main>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      drawer: false,
      loading: false,
      prices: [],
      items: [
        { text: 'Cadastrar Ativos', icon: 'mdi-wallet-bifold', link: '/cadastro' },
        { text: 'Histórico de Preços Ativos', icon: 'mdi-chart-bar', link: '/historico' },
        { text: 'Compra e Venda', icon: 'mdi-currency-usd', link: '/compra_venda' },
        { text: 'Relatório', icon: 'mdi-finance', link: '/relatorio' },
      ],
      headers: [
        { text: 'Ticker', value: 'ticker', class: 'ticker-column' },
        { text: 'Data', value: 'data_ativo', class: 'date-column' },
        { text: 'Abertura', value: 'open', class: 'open-column' },
        { text: 'Mínimo', value: 'low', class: 'low-column' },
        { text: 'Máximo', value: 'high', class: 'high-column' },
        { text: 'Fechamento', value: 'close', class: 'close-column' },
        { text: 'Volume', value: 'volume', class: 'volume-column' }
      ],
      interval: 60000, // Intervalo de 1 minuto
      refreshInterval: null
    };
  },
  mounted() {
    this.fetchData(); // Fetch data immediately on mount
    this.refreshInterval = setInterval(this.fetchData, this.interval); // Atualiza os dados a cada minuto
  },
  beforeUnmount() {
    clearInterval(this.refreshInterval); // Limpa o intervalo ao desmontar o componente
  },
  methods: {
    async fetchData() {
      this.loading = true;
      const token = localStorage.getItem('token'); // Recupera o token do localStorage
      try {
        const response = await axios.get('http://localhost:8000/api/v1/get-stock-history', {
          headers: {
            'Authorization': `Bearer ${token}`
          }
        });
        const data = response.data;
        this.prices = this.processData(data); // Atualize a lista com os dados mais recentes
      } catch (error) {
        console.error("Erro ao buscar dados:", error.response ? error.response.data : error.message);
        this.prices = []; // Limpa os dados em caso de erro
      } finally {
        this.loading = false; // Sempre define loading como false
      }
    },
    processData(data) {
      // Agrupa os dados por ticker e seleciona os registros mais recentes por minuto
      const groupedData = data.reduce((acc, item) => {
        if (!acc[item.ticker]) {
          acc[item.ticker] = [];
        }
        acc[item.ticker].push(item);
        return acc;
      }, {});

      // Ordena os registros dentro de cada grupo e seleciona os mais recentes por minuto
      const processedData = [];
      Object.keys(groupedData).forEach(ticker => {
        const tickerData = groupedData[ticker];
        tickerData.sort((a, b) => new Date(b.data_ativo) - new Date(a.data_ativo)); // Ordena por data

        const uniqueMinutes = new Set();
        tickerData.forEach(item => {
          const minute = new Date(item.data_ativo).getMinutes();
          if (!uniqueMinutes.has(minute)) {
            uniqueMinutes.add(minute);
            processedData.push(item);
          }
        });
      });

      // Limita a 20 tickers únicos e retorna os dados processados
      return processedData.slice(0, 20);
    },
    print() {
      window.print();
    },
    formatValue(value, type) {
      switch (type) {
        case 'currency':
          return `R$ ${parseFloat(value).toFixed(2)}`;
        case 'date':
          return new Date(value).toLocaleString('pt-BR'); // Formata a data para o padrão brasileiro
        default:
          return value;
      }
    },
    toggleDrawer() {
      this.drawer = !this.drawer;
    }
  }
};
</script>

<style scoped>
.elevation-1 {
  background-color: #c0d5e2;
  border-radius: 8px;
  overflow: hidden;
}

.bordered-row {
  border-bottom: 1px solid #0d0344;
}

.cell-center {
  text-align: center;
}

.ticker-column {
  color: #FF4081;
}

.date-column {
  color: #3F51B5;
}

.open-column {
  color: #8BC34A;
}

.low-column {
  color: #FFC107;
}

.high-column {
  color: #F44336;
}

.close-column {
  color: #9C27B0;
}

.volume-column {
  color: #00BCD4;
}

.loading-message {
  text-align: center;
  font-size: 1.5rem;
}
</style>
