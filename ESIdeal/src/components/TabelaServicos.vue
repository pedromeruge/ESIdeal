<template>
  <div class="table-main">
    <div class="table-header" :class="{'red-table-header': comPrazo }">
      <span class="header-text">{{ comPrazo ? 'COM PRAZO LIMITE' : 'SEM PRAZO LIMITE'}}</span>
      <!-- considerar meter os botoes de ordenar e filtrar num componente separado?? -->
      <div class="dropdowns">
        <!-- filtrar dropdown -->
        <div class="filtrar-dropdown">
          <button class="dropdown-button" @click="toggleFiltrarDropdown">FILTRAR
            <img class="dropdown-arrow" src="/svgs/filtrar_arrow.svg"></img>
          </button>
          <div class="dropdown-content" v-show="dropdownFiltroVisible">
            <button v-for="(option,index) in dropdownFiltrarOptions" :class="{ 
              'dropdown-elem': true,
              'dropdown-elem-red': index === filtroSelecionado && comPrazo,
              'dropdown-elem-black': index === filtroSelecionado && !comPrazo
            }"
            @click="applyFiltrarFunc(index,option.function)"> {{ option.title }}</button>
          </div>
        </div>
        <!-- ordenar dropdown -->
        <div class="ordenar-dropdown">
          <button class="dropdown-button" @click="toggleOrdenarDropdown">ORDENAR
            <img class="dropdown-arrow" src="/svgs/filtrar_arrow.svg"></img>
          </button>
          <div class="dropdown-content" v-show="dropdownOrdenarVisible" >
            <template v-for="(option,index) in dropdownOrdenarOptions">
              <button v-if="!('optional' in option) || comPrazo" :class="{ 
                'dropdown-elem': true,
                'dropdown-elem-red': index === ordenarSelecionado && comPrazo,
                'dropdown-elem-black': index === ordenarSelecionado && !comPrazo
              }"
              @click="applyOrdenarFunc(index,option.function)"> {{ option.title }}
            </button>
          </template>
          </div>
        </div>

      </div>
    </div>
    <div class="table-content">
      <div v-for="(service,index) in servicesToPresent" :key="index" class="service">
        <ServicoBanner :servico="service" :Consts="Consts" :ServicesInfo="ServicesInfo"/>
      </div>
    </div>
  </div>
</template>
  
<script>
import ServicoBanner from './ServicoBanner.vue';
import * as Consts from '../models/consts.js';
import * as ServicesInfo from '../models/ServicesInfo.js';

export default {
  components: {
      ServicoBanner
  },

  props: {
    services: {
      type: Array,
      required: true
    },
    comPrazo: {
      type: Boolean,
      required: true
    }
  },

  computed: {
    Consts() {
      return Consts;
    },
    ServicesInfo() {
      return ServicesInfo;
    }
  },

  props: ['services', 'comPrazo'],

  data() {
    return {

      dropdownOrdenarOptions: [ // títulos e funções a chamar para cada botão
        { "title": "Proximidade da data limite", "function": "sortByProxLimit", "optional": true}, // não é para aparecer na tabela de "sem prazo limite"
        { "title": "Ordem de chegada", "function": "sortArrival"},
        { "title": "Ordem crescente de duração", "function": "sortByCrescDuration"},
        { "title": "Ordem decrescente de duração", "function": "sortByDecresDuration"}
      ],
      dropdownFiltrarOptions: [ // títulos e funções a chamar para cada botão

        { "title": "Sem filtro", "function": "filterNone"},
        { "title": "Serviços suspensos", "function": "filterSuspended"},
        { "title": "Serviços combustão", "function": "filterCombustion"},
        { "title": "Serviços elétricos", "function": "filterEletric"},
        { "title": "Serviços universais", "function": "filterUniversal"}
      ],
      dropdownOrdenarVisible: false, // para gerir se dropdown é ou não visível
      dropdownFiltroVisible: false, // para gerir se dropdown é ou não visível
      filtroSelecionado: 0, // posição do elemento dentro da div do dropdown-cotent
      ordenarSelecionado: this.comPrazo ? 0 : 1,
      servicesToPresent: []
    }
  },
  methods: {
    toggleOrdenarDropdown() {
      // find dropdown content element
      this.dropdownOrdenarVisible = !this.dropdownOrdenarVisible;
      this.dropdownFiltroVisible = false
    },
    toggleFiltrarDropdown() {
      // find dropdown content element
      this.dropdownFiltroVisible = !this.dropdownFiltroVisible;
      this.dropdownOrdenarVisible = false
    },

    applyOrdenarFunc(index, func) {
      this.ordenarSelecionado = index;
      this[func]() // correr a função correspondente
      this.dropdownOrdenarVisible = false // fechar o dropdown por conveniencia
    },
    applyFiltrarFunc(index,func) {
      this.filtroSelecionado = index;
      this[func]() // correr a função de filter correspondente  
      const currOrdFunc = this.dropdownOrdenarOptions[this.ordenarSelecionado].function
      this[currOrdFunc]() // correr a função de ordenar correpsondente, sobre dados filtrados
      this.dropdownFiltroVisible = false // fechar o dropdown por conveniencia
    },

    sortArrival() {
      this.servicesToPresent.sort((a,b) => a.id.localeCompare(b.id))
    },
    //funções de ORDENAR
    sortByCrescDuration() {
      this.servicesToPresent.sort((a,b) => a.def_servico.duracao - b.def_servico.duracao)
    },

    sortByDecresDuration() {
      this.servicesToPresent.sort((a,b) => b.def_servico.duracao - a.def_servico.duracao)
    },
    sortByProxLimit() {
      this.servicesToPresent.sort((a, b) => a.data - b.data);
    },

    // funções de FILTER
    filterNone() {
      this.servicesToPresent = this.services
    },
    filterSuspended() {
      this.servicesToPresent = this.services.filter(service => service.estado === Consts.EstadoServico.PARADO);
    },
    filterCombustion() {
      this.servicesToPresent = this.services.filter(service => (service.tipos_servico.includes(Consts.TiposVeiculo.GASOLINA) || service.tipos_servico.includes(Consts.TiposVeiculo.GASOLEO)) && !service.tipos_servico.includes(Consts.TiposVeiculo.ELETRICO));
    },
    filterEletric() {
      this.servicesToPresent = this.services.filter(service => !(service.tipos_servico.includes(Consts.TiposVeiculo.GASOLINA) || service.tipos_servico.includes(Consts.TiposVeiculo.GASOLEO)) && service.tipos_servico.includes(Consts.TiposVeiculo.ELETRICO));
    },
    filterUniversal() {
      this.servicesToPresent = this.services.filter(service => (service.tipos_servico.includes(Consts.TiposVeiculo.GASOLINA) || service.tipos_servico.includes(Consts.TiposVeiculo.GASOLEO)) && service.tipos_servico.includes(Consts.TiposVeiculo.ELETRICO));
    }
  },

  created() {
    this.servicesToPresent = this.services
    if (this.comPrazo) {
      this.sortByProxLimit()
    } else {
      this.sortArrival()
    }
  }
}

</script>

<style scoped>

  .table-main {
    display: flex;
    flex-direction: column;
    width: 100%;
  }

  .table-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    background-color: var(--color-table-black);
    color: white;
    min-width: calc(var(--banner-width-min) - 30px);
    width: calc(100% - 31px);
    min-height: 30px;
    height: fit-content;
    font-family: var(--font-family);
    padding: 0px 5px 0px 15px;
    margin-bottom: 10px;
    font-size: 0.9em;
    font-weight: 400;
  }
  
  .header-text {
    display: flex;
    align-items: center;
  }

  .dropdowns {
    display: flex;
    justify-content: right;
    flex-direction: row;
    align-self: right;
  }
  
  .ordenar-dropdown, .filtrar-dropdown {
    float: right;
    position: relative;
    margin-left: 15px;
  }

  .dropdown-button {
    background-color:  rgba(255, 255, 255,0);
    color: white;
    border: none;
    cursor: pointer;
    font-size: 1em;
    font-weight: 400;
    padding: 2px 5px;
  }

  .dropdown-arrow{
    width: 18px;
    padding-left: 10px;
    translate: 0px 1px;
    pointer-events: none;
  }

  .dropdown-button:hover, .dropdown-button:focus {
    background-color: rgba(255,255,255,0.3);
  }

  .dropdown-content {
    display: block; /* por default não aparece */
    position: absolute;
    background-color: var(--color-light-grey);
    width: 25vw;
    box-shadow: 0px 8px 16px 0px rgba(0,0,0,0.2);
    z-index: 1;
    right: 0;
    translate: 5px 5px;
  }

  .dropdown-elem {
    color: var(--text-black);
    align-items: center;
    text-align: left;
    padding: 4px 10px;
    background-color: var(--color-very-light-grey);
    font-size: 1.3em;
    font-weight: 300;
    min-height: 30px;
    height: fit-content;
    width: 100%;
    display: block;
  }

  .dropdown-elem:not(:last-child) {
    border: solid;
    border-width: 1px 1px 0px 1px;
    border-color: var(--text-light-grey);
  }

  .dropdown-elem:last-child {
    border: solid;
    border-width: 1px;
    border-color: var(--text-light-grey);
  }

  .dropdown-elem:hover {
    background-color: rgba(255,255,255,0.6);
  }

  .table-content {
    overflow-y: scroll;
    width: 100%;
    /* height: 65vh; */
    padding-right: 15px;
  }

  .service:not(:last-child) { /* todos os elementos menos o último da lista ficam com espaço em baixo */
    margin-bottom: 10px;
  }

  /* customizar a scrollbar */
  .table-content::-webkit-scrollbar {
    width: 12px;
  }

  .table-content::-webkit-scrollbar-thumb {
    background-color: #949494;
    border-radius: 10px;
  }

  .table-content::-webkit-scrollbar-track {
    background-color: #E2E2E2;
  }

  /*classes para assign dinâmico*/
  .red-table-header {
    background-color: var(--color-red);
  }

  .dropdown-elem-black {
    background-color: var(--color-table-black);
  }

  .dropdown-elem-red {
    background-color: var(--color-red);
  }

  @media (max-width: 700px) {

    .table-main {
      padding: 0;
      margin: 0;
    }

    .table-header {
      display: flex;
      align-items: flex-start;
      min-width: auto;
      width: calc(100% - 32px);
      min-height: fit-content;
      padding: 5px 10px;

    }

    .table-content {
      padding-right: 10px
    }

    .dropdown-content {
      width: fit-content;
      min-width: 40vw;
    }

    .dropdown-button {
      padding: 0;
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
    }

    .dropdown-button img {
      padding-right: 10px;
    }

    .dropdown-elem {
      padding: 5px 10px;
      min-height: fit-content;
    }
  }

</style>