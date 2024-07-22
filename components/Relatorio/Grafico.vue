<template>
  <div>
    <v-app-bar fixed color="#041022" dark>
      <!-- Ícone para abrir/fechar o drawer -->
      <v-app-bar-nav-icon @click="drawer = !drawer"></v-app-bar-nav-icon>
      
      <!-- Título da aplicação -->
      <v-app-bar-title class="mr-6">Brix Investimentos</v-app-bar-title>
      
      <!-- Botão para carregar arquivos PDF -->
      <v-btn @click="openFilePicker" accept="application/pdf" hide-details class="mr-4">
        Carregar
      </v-btn>
      
      <!-- Dropdown "Início" -->
      <v-menu open-on-hover>
        <template v-slot:activator="{ props }">
          <v-btn color="primary" v-bind="props" class="mr-4">Início: {{ inicio }}</v-btn>
        </template>
        <v-list>
          <v-list-item v-for="option in inicioOptions" :key="option" @click="selectInicio(option)">
            <v-list-item-title>{{ option }}</v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
      
      <!-- Dropdown "Período" -->
      <v-menu open-on-hover>
        <template v-slot:activator="{ props }">
          <v-btn color="primary" v-bind="props" class="mr-4">Período: {{ periodo }}</v-btn>
        </template>
        <v-list>
          <v-list-item v-for="option in periodOptions" :key="option" @click="selectPeriodo(option)">
            <v-list-item-title>{{ option }}</v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
      
      <!-- Dropdown "Conta" -->
      <v-menu open-on-hover>
        <template v-slot:activator="{ props }">
          <v-btn color="primary" v-bind="props" class="mr-4">Conta: {{ conta }}</v-btn>
        </template>
        <v-list>
          <v-list-item v-for="option in accountOptions" :key="option" @click="selectConta(option)">
            <v-list-item-title>{{ option }}</v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
      
      <!-- Dropdown "Estratégia de Alocação" -->
      <v-menu open-on-hover>
        <template v-slot:activator="{ props }">
          <v-btn color="primary" v-bind="props" class="mr-4">Estratégia de Alocação: {{ estrategia }}</v-btn>
        </template>
        <v-list>
          <v-list-item v-for="option in strategyOptions" :key="option" @click="selectEstrategia(option)">
            <v-list-item-title>{{ option }}</v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
      
      <!-- Dropdown "Grupo de Estratégia" -->
      <v-menu open-on-hover>
        <template v-slot:activator="{ props }">
          <v-btn color="primary" v-bind="props">Grupo de Estratégia: {{ grupo }}</v-btn>
        </template>
        <v-list>
          <v-list-item v-for="option in groupOptions" :key="option" @click="selectGrupo(option)">
            <v-list-item-title>{{ option }}</v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
      
      <!-- Ícone de Impressora -->
      <v-icon @click="print" class="ml-6">mdi-printer</v-icon>
      
      <!-- Input file hidden para carregar arquivos -->
      <input
        ref="fileInput"
        type="file"
        style="display: none;"
        accept="application/pdf"
        @change="handleFileUpload"
      />
    </v-app-bar>

    <v-navigation-drawer v-model="drawer" temporary clipped app>
      <!-- Conteúdo do drawer aqui -->
      <v-list dense>
        <v-list-item v-for="item in items" :key="item.text" :to="{ path: item.link }">
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
        <!-- Cards de informações -->
        <v-row>
          <v-col cols="12" md="6" lg="4" v-for="card in infoCards" :key="card.title">
            <v-card class="pa-4">
              <div>
                <h3>{{ card.title }}</h3>
                <p>{{ card.value }}</p>
              </div>
            </v-card>
          </v-col>
        </v-row>

        <!-- Gráficos -->
        <v-row>
          <v-col cols="12" md="6">
            <v-card class="pa-4">
              <line-chart :chart-data="rentabilidadeData" />
            </v-card>
          </v-col>
          <v-col cols="12" md="6">
            <v-card class="pa-4">
              <bar-chart :chart-data="rentabilidadeMensalData" />
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </div>
</template>


<script>
import { Line, Bar } from 'vue-chartjs'
import { Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale, PointElement, LineElement } from 'chart.js'
ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale, PointElement, LineElement)

export default {
  data() {
    return {
      drawer: false,
      inicio: "Hoje",
      inicioOptions: ["Hoje", "Ontem", "Semana Passada", "Mês Passado"],
      periodo: "6 meses",
      periodOptions: ["1 mês", "3 meses", "6 meses", "1 ano"],
      conta: "Todas as contas",
      accountOptions: ["Todas as contas", "Conta específica 1", "Conta específica 2", "Conta específica 3"],
      estrategia: "Tipo de Ativo",
      strategyOptions: ["Ações", "Renda Fixa", "Fundos Imobiliários"],
      grupo: "Ações",
      groupOptions: ["Ticker ABC", "Ticker XYZ", "Ticker QRS"],
      items: [
        { text: 'Cadastrar Ativos', icon: 'mdi-wallet-bifold', link: '/cadastro' },
        { text: 'Histórico de Preços Ativos', icon: 'mdi-chart-bar', link: '/historico' },
        { text: 'Compra e Venda', icon: 'mdi-currency-usd', link: '/compra_venda' },
        { text: 'Relatório', icon: 'mdi-finance', link: '/relatorio' },
      ],
      infoCards: [],
      rentabilidadeData: {},
      rentabilidadeMensalData: {},
    };
  },
  methods: {
    openFilePicker() {
      this.$refs.fileInput.click();
    },
    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file.type === 'application/pdf') {
        console.log('Arquivo carregado:', file);
      } else {
        console.error('Por favor, selecione um arquivo PDF válido.');
      }
    },
    print() {
      window.print();
    },
    selectInicio(option) {
      this.inicio = option;
    },
    selectPeriodo(option) {
      this.periodo = option;
    },
    selectConta(option) {
      this.conta = option;
    },
    selectEstrategia(option) {
      this.estrategia = option;
    },
    selectGrupo(option) {
      this.grupo = option;
    },
    async generateReport() {
      try {
        const response = await axios.post('/api/relatorio-diversos', {
          inicio: this.inicio,
          periodo: this.periodo,
          conta: this.conta,
          estrategia: this.estrategia,
          grupo: this.grupo
        });
        this.processReport(response.data);
      } catch (error) {
        console.error('Erro ao gerar relatório:', error);
      }
    },
    processReport(data) {
      this.infoCards = data.map((item, index) => ({
        title: `Relatório ${index + 1}`,
        value: `Posição Inicial: R$ ${item.posicao_inicial.toFixed(2)}, Posição Final: R$ ${item.posicao_final.toFixed(2)}, Movimentação: R$ ${item.movimentacao.toFixed(2)}, Rentabilidade: ${item.rentabilidade_periodo.toFixed(2)}%, Volatilidade: ${item.volatilidade.toFixed(2)}%, Resultado Projetado: R$ ${item.resultado_projetado.toFixed(2)}`
      }));
      // Gerar dados para gráficos
      this.rentabilidadeData = {
        labels: data.map((_, index) => `Relatório ${index + 1}`),
        datasets: [
          {
            label: 'Rentabilidade',
            backgroundColor: '#f87979',
            data: data.map(item => item.rentabilidade_periodo)
          }
        ]
      };
      this.rentabilidadeMensalData = {
        labels: data.map((_, index) => `Relatório ${index + 1}`),
        datasets: [
          {
            label: 'Rentabilidade Mensal',
            backgroundColor: '#f87979',
            data: data.map(item => item.rentabilidade_periodo / 6) // Exemplo de cálculo mensal
          }
        ]
      };
    }
  }
};
</script>

<style scoped>
.v-app-bar {
  background-color: #041022; /* Cor de fundo */
}
.v-btn--text {
  color: #4A3AFF; /* Cor do texto dos botões */
}
.v-file-input input {
  display: none; /* Esconder o input de arquivo padrão */
}
.v-container {
  margin-top: 16px; /* Espaçamento superior */
}
</style>

