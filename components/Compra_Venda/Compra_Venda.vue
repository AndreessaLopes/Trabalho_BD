<template>
  <!-- App Bar e Drawer -->
  <v-app-bar fixed color="#041022" dark>
    <v-app-bar-nav-icon @click="toggleDrawer"></v-app-bar-nav-icon>
    <v-app-bar-title class="mr-6">Compra e Venda de Ativos</v-app-bar-title>
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

  <!-- Main Content -->
  <v-main>
    <v-container>
      <!-- Ativos Disponíveis e Detalhes do Ativo -->
      <v-row>
        <v-col cols="12" md="6">
          <v-card class="mb-4">
            <v-card-title class="primary white--text">Ativos Disponíveis</v-card-title>
            <v-card-text>
              <v-data-table
                :headers="headers"
                :items="ativos"
                item-key="ticker"
                class="elevation-1"
              >
                <template v-slot:item="{ item }">
                  <tr @click="selectAtivo(item)" class="cursor-pointer">
                    <td class="text-start">{{ item.ticker }}</td>
                    <td>{{ item.nome }}</td>
                    <td class="text-end">{{ formatValue(item.precoAtual, 'currency') }}</td>
                  </tr>
                </template>
              </v-data-table>
            </v-card-text>
          </v-card>
        </v-col>
        <v-col cols="12" md="6">
          <v-card class="mb-4" v-if="ativoSelecionado">
            <v-card-title class="accent white--text">Detalhes do Ativo</v-card-title>
            <v-card-text>
              <div>
                <span class="info-label">Ticker:</span> {{ ativoSelecionado.ticker }}
              </div>
              <div>
                <span class="info-label">Nome:</span> {{ ativoSelecionado.nome }}
              </div>
              <div>
                <span class="info-label">Tipo:</span> {{ ativoSelecionado.tipo }}
              </div>
              <div>
                <span class="info-label">Preço Atual:</span> {{ formatValue(ativoSelecionado.precoAtual, 'currency') }}
              </div>
              <div v-if="ativoSelecionado.setor">
                <span class="info-label">Setor:</span> {{ ativoSelecionado.setor }}
              </div>
              <div v-if="ativoSelecionado.industria">
                <span class="info-label">Indústria:</span> {{ ativoSelecionado.industria }}
              </div>
            </v-card-text>
            <v-card-actions>
              <v-btn color="primary darken-2" @click="openForm('compra')">Comprar</v-btn>
              <v-btn color="error darken-2" @click="openForm('venda')">Vender</v-btn>
            </v-card-actions>
          </v-card>
        </v-col>
      </v-row>

      <!-- Formulário de Compra/Venda -->
      <v-dialog v-model="dialog" max-width="600">
        <v-card>
          <v-card-title>{{ formMode === 'compra' ? 'Compra de Ativo' : 'Venda de Ativo' }}</v-card-title>
          <v-card-text>
            <v-form @submit.prevent="submitAporte">
              <v-text-field v-model="form.quantidade" label="Quantidade" type="number" required></v-text-field>
              <v-text-field v-model="form.precoUnitario" label="Preço Unitário" type="number" required></v-text-field>
              <v-btn type="submit" color="primary">Confirmar</v-btn>
            </v-form>
          </v-card-text>
        </v-card>
      </v-dialog>

      <!-- Compras e Vendas Realizadas -->
      <v-card class="mb-4">
        <v-card-title class="accent white--text">Minhas Compras e Vendas</v-card-title>
        <v-card-text>
          <v-data-table
            :headers="transacaoHeaders"
            :items="transacoes"
            class="elevation-1"
          >
            <template v-slot:item="{ item }">
              <tr>
                <td>{{ item.tipo }}</td>
                <td>{{ item.ticker }}</td>
                <td>{{ item.quantidade }}</td>
                <td>{{ formatValue(item.precoUnitario, 'currency') }}</td>
                <td>{{ formatValue(item.valorTotal, 'currency') }}</td>
                <td>{{ formatValue(item.data, 'date') }}</td>
              </tr>
            </template>
          </v-data-table>
        </v-card-text>
      </v-card>
    </v-container>
  </v-main>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      drawer: false,
      ativos: [],
      items: [
        { text: 'Cadastrar Ativos', icon: 'mdi-wallet-bifold', link: '/cadastro' },
        { text: 'Histórico de Preços', icon: 'mdi-chart-bar', link: '/historico' },
        { text: 'Relatório', icon: 'mdi-finance', link: '/relatorio' }
      ],
      headers: [
        { text: 'Ticker', value: 'ticker' },
        { text: 'Nome', value: 'nome' },
        { text: 'Preço Atual', value: 'precoAtual' }
      ],
      transacaoHeaders: [
        { text: 'Tipo', value: 'tipo' },
        { text: 'Ticker', value: 'ticker' },
        { text: 'Quantidade', value: 'quantidade' },
        { text: 'Preço Unitário', value: 'precoUnitario' },
        { text: 'Valor Total', value: 'valorTotal' },
        { text: 'Data', value: 'data' }
      ],
      ativoSelecionado: null,
      dialog: false,
      formMode: '',
      form: {
        quantidade: null,
        precoUnitario: null
      },
      transacoes: []
    };
  },
  async mounted() {
    // Gerar e atualizar ativos iniciais ao montar o componente
    await this.gerarEAtualizarAtivos();
    // Iniciar intervalo para atualizar ativos a cada 10 segundos
    setInterval(this.atualizarAtivos, 10000);
  },
  methods: {
    async gerarEAtualizarAtivos() {
      const tickers = ['AAPL', 'GOOGL', 'MSFT', 'AMZN', 'TSLA', 'NVDA', 'NFLX', 'META', 'V', 'JPM'];
      const apiKey = 'SUA_CHAVE_DE_API_ALPHA_VANTAGE'; // Substitua pela sua chave de API

      for (let i = 0; i < tickers.length; i++) {
        const randomTicker = tickers[i];
        try {
          const response = await axios.get(`https://www.alphavantage.co/query`, {
            params: {
              function: 'TIME_SERIES_INTRADAY',
              symbol: randomTicker,
              interval: '1min',
              apikey: apiKey
            }
          });

          const data = response.data['Time Series (1min)'];
          const lastRefreshed = Object.keys(data)[0];
          const lastPrice = data[lastRefreshed]['4. close'];

          const novoAtivo = {
            ticker: randomTicker,
            nome: randomTicker, // Nome fictício, pois Alpha Vantage pode não fornecer
            tipo: 'Tecnologia', // Pode ajustar conforme necessário
            precoAtual: lastPrice,
            setor: 'Tecnologia', // Ajuste conforme necessário
            industria: 'Software' // Ajuste conforme necessário
          };

          this.ativos.push(novoAtivo);
        } catch (error) {
          console.error('Erro ao gerar ativo aleatório:', error);
          // Tratar erros de forma adequada
        }
      }
    },
    async atualizarAtivos() {
      const apiKey = 'SUA_CHAVE_DE_API_ALPHA_VANTAGE'; // Substitua pela sua chave de API

      for (let i = 0; i < this.ativos.length; i++) {
        const ativo = this.ativos[i];
        try {
          const response = await axios.get(`https://www.alphavantage.co/query`, {
            params: {
              function: 'TIME_SERIES_INTRADAY',
              symbol: ativo.ticker,
              interval: '1min',
              apikey: apiKey
            }
          });

          const data = response.data['Time Series (1min)'];
          const lastRefreshed = Object.keys(data)[0];
          const lastPrice = data[lastRefreshed]['4. close'];

          ativo.precoAtual = lastPrice;
        } catch (error) {
          console.error('Erro ao atualizar dados do ativo:', error);
          // Tratar erros de forma adequada
        }
      }
    },
    selectAtivo(ativo) {
      this.ativoSelecionado = ativo;
    },
    openForm(mode) {
      this.formMode = mode;
      this.dialog = true;
    },
    submitAporte() {
      // Simular ação de realizar aporte
      const tipoTransacao = this.formMode === 'compra' ? 'Compra' : 'Venda';
      const valorTotal = this.form.quantidade * this.form.precoUnitario;
      const dataAtual = new Date().toISOString().split('T')[0];

      this.transacoes.unshift({
        tipo: tipoTransacao,
        ticker: this.ativoSelecionado.ticker,
        quantidade: this.form.quantidade,
        precoUnitario: this.form.precoUnitario,
        valorTotal: valorTotal,
        data: dataAtual
      });

      // Lógica adicional para enviar os dados para API ou processar localmente
      this.dialog = false;
      this.form.quantidade = null;
      this.form.precoUnitario = null;
    },
    print() {
      window.print();
    },
    toggleDrawer() {
      this.drawer = !this.drawer;
    },
    formatValue(value, type) {
      switch (type) {
        case 'currency':
          return `R$ ${parseFloat(value).toFixed(2)}`;
        case 'date':
          return new Date(value).toLocaleDateString();
        default:
          return value;
      }
    }
  }
};
</script>

<style scoped>
.cursor-pointer {
  cursor: pointer;
}
.primary {
  background-color: #041022;
}
.accent {
  background-color: #FFC107;
}
.white--text {
  color: #ffffff;
}
</style>
