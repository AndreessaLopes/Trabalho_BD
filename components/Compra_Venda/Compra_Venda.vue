<template>
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
    await this.fetchAtivos();
    await this.fetchTransacoes();  // Fetch transacoes initially
    this.startAtivosUpdateInterval();
  },
  methods: {
    toggleDrawer() {
      this.drawer = !this.drawer;
    },
    async fetchAtivos() {
      const token = localStorage.getItem('token');
      try {
        const response = await axios.get('http://localhost:8000/api/v1/cadastrar-ativos', {
          headers: {
            'Authorization': `Bearer ${token}`
          }
        });
        this.ativos = response.data.map(item => ({
          ...item,
          precoAtual: parseFloat(item.precoAtual) || 0  // Garantir que o valor é um número
        }));
      } catch (error) {
        console.error('Erro ao buscar ativos:', error);
      }
    },
    async fetchTransacoes() {
      const token = localStorage.getItem('token');
      try {
        const [comprasResponse, vendasResponse] = await Promise.all([
          axios.get('http://localhost:8000/api/v1/compras', {
            headers: {
              'Authorization': `Bearer ${token}`
            }
          }),
          axios.get('http://localhost:8000/api/v1/vendas', {
            headers: {
              'Authorization': `Bearer ${token}`
            }
          })
        ]);

        const formatTransacoes = (items, tipo) => items.map(item => ({
          ...item,
          tipo,
          valorTotal: parseFloat(item.valor_total) || 0,  // Garantir que o valor é um número
          precoUnitario: parseFloat(item.valor_unitario) || 0,  // Garantir que o valor é um número
          data: new Date(item.data).toLocaleDateString('pt-BR') // Formatar data
        }));

        const compras = formatTransacoes(comprasResponse.data, 'Compra');
        const vendas = formatTransacoes(vendasResponse.data, 'Venda');

        this.transacoes = [...compras, ...vendas];
      } catch (error) {
        console.error('Erro ao buscar transações:', error);
      }
    },
    startAtivosUpdateInterval() {
      setInterval(() => {
        this.fetchAtivos();
      }, 10000);
    },
    selectAtivo(ativo) {
      this.ativoSelecionado = ativo;
    },
    openForm(mode) {
      this.formMode = mode;
      this.dialog = true;
    },
    async submitAporte() {
      const token = localStorage.getItem('token');
      try {
        const url = this.formMode === 'compra'
          ? 'http://localhost:8000/api/v1/compras'
          : 'http://localhost:8000/api/v1/vendas';

          await axios.post(url, {
          id_ticker: this.ativoSelecionado.id, // Ajustado para 'id_ticker'
          quantidade: parseInt(this.form.quantidade, 10), // Garantir que seja um número inteiro
          valor_unitario: parseFloat(this.form.precoUnitario), // Garantir que seja um número decimal
          valor_total: parseFloat(this.form.quantidade) * parseFloat(this.form.precoUnitario) // Calcular valor total
        }, {
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'  // Adicione este header se o backend esperar JSON
          }
        });

        this.dialog = false;
        this.fetchTransacoes(); // Atualizar transações após o envio
      } catch (error) {
        console.error('Erro ao enviar o formulário:', error);
      }
    },
    formatValue(value, type) {
      if (type === 'currency') {
        return new Intl.NumberFormat('pt-BR', {
          style: 'currency',
          currency: 'BRL'
        }).format(value);
      }
      if (type === 'date') {
        return new Date(value).toLocaleDateString('pt-BR');
      }
      return value;
    },
    print() {
      window.print();
    }
  }
};
</script>

<style scoped>
.info-label {
  font-weight: bold;
  color: #333;
}
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
