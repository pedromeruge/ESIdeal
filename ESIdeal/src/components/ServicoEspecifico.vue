<script>
	import ModalConc from './ModalConc.vue';
    import Navbar from './Navbar.vue';
    import Clock from './Clock.vue';
    import ModalSusp from './ModalSusp.vue';
    import Help from './Help.vue';
    import {serviceState} from '../scripts/stores.js';
    import { userState } from '../scripts/stores.js';
    import * as Consts from '../models/consts.js';
    import * as DBRequests from '../scripts/DBrequests.js';
    import PopStart from './PopStart.vue'
    import PopConfirmar from './PopConfirmar.vue';

    export default {
        components: {
            Navbar,
            Clock,
			ModalConc,
            ModalSusp,
            Help,
            PopStart,
            PopConfirmar
        },
        props: {
            servicoID: { // recebido no URL
                type: String,
                required: true
            }
        },

        data() {
            return {
                user: userState(),
                
                servico: null,
                
                horaFim: "",

                sortColumn: null,
                sortOrder: null,

                mostrarStartPopup: false,
				mostrarMenuSuspender: false,
				mostrarMenuConcluir: false,

                //popup confirmar
				mostrarConfirmacaoPopup: false,
                estadoConfirmacaoPopup: "", // se é para suspender ou concluir
                sucessoConfirmacao: false, // se correu bem ou mal
            }
        },

        mounted() {
            this.checkUser();
        },

		methods: {
            checkUser() {
                if (!this.user.userId) {
                    this.$router.push('/login');
                }
            },

            calculateTime(dataFim) {
                var monthNames = ["Janeiro","Fevereiro","Março","Abril","Maio","Junho", "Julho", "Agosto", "Setembro", "Outubro","Novembro","Dezembro"]

                const horaFim = dataFim.getHours().toString().padStart(2, '0');
                const minutosFim = dataFim.getMinutes().toString().padStart(2, '0');
                this.horaFim = `${horaFim}:${minutosFim}h`;

                const currentTime = new Date();
                const diffMinutes = Math.floor((dataFim - currentTime) / 60000);
                const diffDays = Math.floor(diffMinutes / (60 * 24))

                if (diffMinutes < 0 || diffDays > 0) {
                    const diaFim = dataFim.getDate().toString().padStart(2,"0");
                    const mesFim = monthNames[dataFim.getMonth()]
                    this.horaFim = this.horaFim + " (" + diaFim + " " + mesFim + ")"
                }
            },

            goToService(serviceId) {
                this.$router.push('/servico/' + serviceId)
            },

			showModalConc() {
				this.mostrarMenuConcluir = true;
			},
			closeModalConc() {
				this.mostrarMenuConcluir = false;
			},
            popupConc(sucesso) {
                this.closeModalConc();
                this.mostrarConfirmacaoPopup = true;
                this.estadoConfirmacaoPopup = Consts.EstadoServico.REALIZADO
                this.sucessoConfirmacao = sucesso
            },
            showModalSusp() {
				this.mostrarMenuSuspender = true;
			},
			closeModalSusp() {
				this.mostrarMenuSuspender = false;
			},
            popupSusp(sucesso) {
                this.closeModalSusp();
                this.mostrarConfirmacaoPopup = true;
                this.estadoConfirmacaoPopup = Consts.EstadoServico.PARADO
                this.sucessoConfirmacao = sucesso;
            },
            showStartPopup() {
			    this.mostrarStartPopup = true;
            },
            closeStartPopup() {
                this.mostrarStartPopup = false;
            },

            getEstadoFromType(estadoENUM){
                const estado = Consts.getTipoEstadoServico(estadoENUM);
                return estado;
            },

            sortTable(coluna) {
                // ao clicar outra vez trocar entre asc/desc;
                if (this.sortColumn === coluna) {
                    if (this.sortOrder === 'asc') {
                        this.sortOrder = 'desc';
                    } else {
                        this.sortOrder = 'asc';
                    }
                } else {
                    this.sortColumn = coluna;
                    this.sortOrder = 'asc';
                }
                // sort na lista dos historicos
                this.servico.historico.sort((a,b) => {
                    let val1;
                    let val2;
                    if (coluna == 'servico'){
                        val1 = a.def_servico.descricao;
                        val2 = b.def_servico.descricao; 
                    }
                    else{
                        val1 = a[coluna];
                        val2 = b[coluna];
                    }

                    if(coluna === 'data') {
                        val1 = new Date(val1).getTime();
                        val2 = new Date(val2).getTime();
                    }

                    else if(coluna == 'estado') {
                        val1 = this.getEstadoFromType(val1);
                        val2 = this.getEstadoFromType(val2);
                    }

                    console.log("sort valores -> " + val1 + " " + val2);

                    let result;
                    if (val1 < val2){
                        result = -1;
                    }
                    else if (val1 === val2){
                        result = 0;
                    }
                    else {
                        result = 1;
                    }

                    if (this.sortOrder === 'asc'){
                        return result;
                    }
                    else{
                        return -result;
                    }
                });
                console.log(`Sort ${coluna} em ${this.sortOrder}`);
            },
            isGasolina(servico){
                return servico.veiculo.tipo === Consts.TiposVeiculo.GASOLINA;
            },
            isGasoleo(servico){
                return servico.veiculo.tipo === Consts.TiposVeiculo.GASOLEO;
            },
            isHibrido(servico){
                return servico.veiculo.tipo === Consts.TiposVeiculo.HIBRIDO;
            },
            isEletrico(servico){
                return servico.veiculo.tipo === Consts.TiposVeiculo.ELETRICO;
            },

            async startService(){
                // da update na db mas nao pega no estado que estava na db
                const result = await DBRequests.postStartedService(this.servico.id)
                if (result) {
                    this.servico.estado = Consts.EstadoServico.ADECORRER

                    //indicar serviço a decorrer no registo de estado
                    const dbData = serviceState();
                    await dbData.addOnGoingService(this.servico.id)
                    
                    dbData.updateServiceState(this.servico.id, Consts.EstadoServico.ADECORRER)
                }
            },
		},

        computed:{

            anyOngoingService(){
                const store = serviceState();
                store.reloadServicesToCompleteDBdata()
                const onGoingService = store.onGoingService;

                return (onGoingService == null)
            },

            servicoADecorrer() {
                return this.servico.estado === Consts.EstadoServico.ADECORRER
            },

            servicoParado() {
                return this.servico.estado === Consts.EstadoServico.PARADO
            },

            servicoConcluido() {
                return this.servico.estado === Consts.EstadoServico.REALIZADO
            },

            matricula(){
                try{    
                    const matricula = this.servico.veiculo.id;
                    return matricula;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },

            marca(){
                try{    
                    const marca = this.servico.veiculo.marca;
                    return marca;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            modelo(){
                try{    
                    const modelo = this.servico.veiculo.modelo;
                    return modelo;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            cilindrada(){
                try{    
                    const cilindrada = this.servico.veiculo.cilindrada;
                    return cilindrada;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            motor(){
                try{    
                    const motor = this.servico.veiculo.motor;
                    return motor;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            potencia(){
                try{    
                    const potencia = this.servico.veiculo.potencia;
                    return potencia;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            medidasJantes(){
                try{    
                    const medidasJantes = this.servico.veiculo.medidasJantes;
                    return medidasJantes;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            tipoMotor(){
                try{    
                    const tipoMotor = Consts.getTipoVeiculoString(this.servico.veiculo.tipo);
                    return tipoMotor;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            nomecliente(){
                try{    
                    const nomecliente = this.servico.cliente.nome;
                    return nomecliente;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            contacto(){
                try{    
                    const contacto = this.servico.cliente.telefone;
                    return contacto;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            estado(){
                try{    
                    const estado = Consts.getTipoEstadoServico(this.servico.estado);
                    return estado;
                } catch(e){
                    console.log(e)
                    return 1;
                }
            },
            poucoTempo() {
                const serviceTime = this.servico.data
                const currentTime = Date.now();
                const diffMinutes = Math.floor((serviceTime - currentTime) / 60000);
                return diffMinutes < 60;
            },
        },

        async created() {
            const dbData = serviceState();
            this.servico = await dbData.getServiceDetails(this.servicoID);
            if (this.servico.agendamento === Consts.AgendamentoServico.PROGRAMADO) {
                this.calculateTime(this.servico.data)
            }
        },
    };

 </script>

<template>
    <Navbar linkBackTo="/atribuidos"/>
    <!-- test if data loaded -->
    <div v-if="servico" class="page" :class="{ 'page-shadow' : servicoADecorrer }"> 
        <div class="header">
            <div class="title">
                <h1>Detalhes de serviço</h1>
                <Help hoverIMG="/images/Tooltip.png" />
            </div>
            <Clock class="clock"/>
        </div>
        
        <!-- Banner -->
        <div class="banner">
            <div class="rectangle" :style="{ backgroundColor: poucoTempo ? 'var(--color-red)' : 'var(--color-darker-grey)' }">
                <img v-if="estado === 'PARADO'" src="/svgs/paused.svg" alt="suspenso" class="suspenso-image">
            </div>
            <div class="info" :class="{ 'banner-shadow' : servicoADecorrer }">
                <div class="left">
                    <span class="id">{{servico.def_servico.descricao}} (#{{ servico.id }})</span>
                    <span class="descricao">Tarefa: {{servico.descricao_especifica}}</span>
                    <span class="duracao">Duração: {{ servico.def_servico.duracao }} min</span>
                </div>
                <div class="right">
                    <span class="estado">ESTADO: {{ estado }}</span>
                    <span class="prazo" v-if="servico.agendamento === 2" :class="{'red-text': poucoTempo}">POR TERMINAR ÀS {{ horaFim }}</span>
                    <span class="prazo" v-else>SEM PRAZO LIMITE</span>
                </div>
            </div>
        </div>
        
        <div class="details">
            <!-- Car info -->
            <div class="car-details">
                <span class="veiculo"> Veículo </span>
                <span class="car-info"> Matricula: {{ servico.veiculo.id }} </span>
                <span class="car-info"> Marca: {{ servico.veiculo.marca }} </span>
                <span class="car-info"> Modelo: {{ servico.veiculo.modelo }} </span>
                <span class="car-info" v-if="tipoMotor === 'Gasolina' || tipoMotor === 'Gasóleo'"> Cilindrada: {{ cilindrada }} V</span>
                <span class="car-info" v-if="tipoMotor === 'Elétrico' || tipoMotor === 'Híbrido'"> Potência do carregador: {{ potencia }}</span>
                <span class="car-info"> Medidas Jantes: {{ medidasJantes }}</span>
            </div>

            <div class="separator-phone"></div>

            <!-- Motor info -->
            <div class="motor">
                <span class="tipo-motor"> Tipo de Motor: {{ tipoMotor }}</span>
                <div class="motor-imgs">
                    <!-- Motor combustão -->
                    <img v-if="tipoMotor === 'Gasolina' || tipoMotor === 'Gasóleo'" src="/images/combustao_naoselected.png" alt="combustao">
                    <!-- é suposto ter gasolina e gasóleo ou combustão só? -->
                    <!-- Motor Elétrico -->
                    <img v-else-if="tipoMotor === 'Elétrico'" src="/images/eletrico_naoselected.png" alt="eletrico">

                    <!-- Motor Híbrido -->
                    <span class="motor-imgs" v-else-if="tipoMotor === 'Híbrido'">
                        <img src="/images/combustao_naoselected.png" alt="combustao">
                        <img src="/images/eletrico_naoselected.png" alt="eletrico">
                    </span>
                </div>
            </div>

            <div class="separator"></div>

            <div class="separator-phone"></div>

            <!-- Client Info-->
            <div class="cliente">
                <div class="cliente-title"> Cliente </div>
                <span class="client-info"> Nome: {{ nomecliente }}</span>
                <span class="client-info"> Contacto: {{ contacto }}</span>
            </div>
        </div>

        <!-- razao suspensao -->
        <div v-if="estado === 'PARADO'" class="status">
            <hr>
            <div class="status-message">
                <h2>Motivo Suspensão</h2>
                <p> {{ servico.detalhes.razao_suspensao }}</p>
            </div>
            <hr>
        </div>

        <!-- serviço concluido -->
        <div v-if="estado === 'REALIZADO'" class="status">
            <hr>
            <div class="status-message">
                <h2>Conclusão</h2>
                <p> {{ servico.detalhes.notas_concluido }}</p>
                <p> {{ servico.detalhes.data_conclusao }}</p>
            </div>
            <hr>
            
        </div>

        <!-- Tabela de serviços anteriores -->
        <div class="table">
            <div class="table-container">
                <div class="table-header">
                    <span>HISTÓRICO DE SERVIÇOS DO CARRO</span>
                </div>
                <div class="table-services">
                    <table>
                        <thead>
                            <tr>
                                <th @click="sortTable('servico')">Serviço</th>
                                <th @click="sortTable('estado')">Estado</th>
                                <th @click="sortTable('data')">Data</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="(service, index) in servico.historico" :key="index" @click="goToService(service.id)" style="cursor: pointer;">
                                <td>{{ service.def_servico.descricao }}</td>
                                <td>{{ getEstadoFromType(service.estado) }}</td>
                                <td>{{ service.estado === 5 ? service.data.toLocaleDateString() : '-' }}</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
        
        <!-- de forma a funcionar como antes usar @click="startService" enves do popup -->
        <button  @click="showStartPopup" v-show="(!servicoADecorrer && !servicoConcluido && anyOngoingService)" class="floating-button">
            INICIAR
            <img class="right-arrow" src="/svgs/back_arrow.svg"></img>
        </button>
        <PopStart v-show="mostrarStartPopup" :currService="this.servico" @close="closeStartPopup"/>

        <div class="buttons">
            <button v-show="(!mostrarMenuConcluir && !mostrarMenuSuspender) && servicoADecorrer" @click="showModalSusp" class="suspend-button">
                SUSPENDER
                <img class="right-arrow" src="/svgs/paused.svg"></img>
            </button>
            <ModalSusp v-show="mostrarMenuSuspender" @close="closeModalSusp" @suspend="popupSusp" :currentService="this.servico"/>
            
            <button v-show="(!mostrarMenuConcluir && !mostrarMenuSuspender) && servicoADecorrer" @click="showModalConc" class="floating-button">
                CONCLUIR
                <img class="right-arrow" src="/svgs/back_arrow.svg"></img>
            </button>
        </div>

		<ModalConc v-show="mostrarMenuConcluir" @close="closeModalConc" @finish="popupConc" :currentService="this.servico"/>

        <PopConfirmar v-if="mostrarConfirmacaoPopup" :tipoEstado="estadoConfirmacaoPopup" :sucesso="sucessoConfirmacao"></PopConfirmar>

    </div>
    <LoadingPage v-else/>
    <Footer/>
</template>

<style scoped>

    .page{
        font-size: 1em;
    }

    .header{
        display: flex;
        flex-direction: row;
        justify-content: space-between;
        align-items: center;
        margin: 0 2.4vw
    }

    .title{
        display: flex;
        align-items: center;
        justify-content: start;
        gap: 20px;
    }

    .info {
        display: flex;
        justify-content: space-between;
        flex-direction: row;
        font-family: var(--font-family);
        width: 100%;
    }

    .banner {
        background-color: var(--color-light-grey);
        display: flex;
        flex-direction: row;
        margin-left: auto;
        margin-right: auto;
        width: 95vw;
    }

    .rectangle {
        display: flex;
        align-items: center;
        justify-content: center;
        /*background-color: var(--color-darker-grey);  */  
        width: 45px;
    }

    .info {
        padding: 5px;
    }
    .rectangle .suspenso-image {
        width: 80%; 
        height: auto;
        display: block;
    }

    .left {
        display: flex;
        flex-direction: column;
        justify-content: flex-start;
        padding: 5px 0px 0px 15px
    }

    .id {
        font-size: 1.875em;
        font-weight: 550;
    }

    .descricao {
        font-size: 1.25em;
        font-weight: 500;
    }

    .right {
        display: flex;
        flex-direction: column;
        align-items: end;
        flex: 2.4;
        padding: 15px 50px 0px 0px
    }

    .prazo {
        display: flex;
        flex:1;
        font-size: 1.1em;
        font-weight: 400;
        color: var(--text-darker-grey);
        margin-top: auto;
        padding-bottom: 20px;
        align-items: flex-end;
   }

    .estado {
        font-size: 1.56em;
        font-weight: 550;
        text-align: right;
    }

    .descricao {
        padding-top: 5px;
    }
    .duracao {
        font-size: 1.12em;
        font-weight: 500;
        padding-top: 10px;
    }

    .details {
        display: flex;
        flex-direction: row;
        padding: 20px 0px;
        align-items: stretch;
        margin: 0 2.4vw
    }

    .car-details, .client-info {
        display: flex;
        flex-direction: column;
    }

    .veiculo, .cliente-title {
        font-size: 2.8em;
        padding-bottom: 30px;
    }

    .car-info, .client-info  {
        padding-bottom: 25px;
        font-size: 1.87em;
    }

    .motor {
        display: flex;
        flex-direction: column;
        align-items: center;
        padding: 40px 0px 0px 80px
    }

    .tipo-motor {
        font-size: 1.87em;
        padding-bottom: 10px;
        text-align: center;
    }

    .motor-imgs {
        display: flex;
        align-items: center;
        justify-content: center;
        flex-wrap: wrap;
    }
    .motor-imgs img{
        width: 80px;
        height: auto;
        padding: 5px 10px;
    }

    .separator {
        margin-left: 50px;
        margin-right: 50px;
        width: 2px;
        background-color: rgb(194, 181, 181);
    }

    .table-container {
        width: 95%;
        margin: 0 auto;
        padding-bottom: 50px;
    }

    .table-header {
        color: white;
        background-color: var(--color-red);
        height: 40px;
        font-family: var(--font-family);
        margin-top: 40px;
        padding: 10px 0px 0px 10px;
        font-size: 1.56em;
        margin-right: 12px;
    }

    .table-services {
        width: 100%; 
        max-height: 500px;
        overflow-y: scroll;
        padding-right: 15px;
    }

    .table-services table {
        width: 100%;
        border-collapse: collapse;
    }

    .table-services th {
        border: 1px solid #ccc;
        background-color: white;
        color: black;
        padding: 10px;
        text-align: left;
        font-size: 1.56em;
        cursor: pointer;
        transition: background-color 0.5s;
    }

    .table-services th:hover {
        background-color: #f2f2f2;
    }
    
    .table-services td {
        padding: 10px;
        text-align: left;
        border: 1px solid #ccc;
        font-size: 1.56em;
    }

    .table-services th:nth-child(1) {
        width: 50%;  /* Coluna Estado */
    }

    .table-services th:nth-child(2) {
        width: 20%;  /* Coluna data */
    }

    .table-services th:nth-child(3) {
        width: 30%;  /* Coluna Data */
    }

    .table-services tr:nth-child(even) {
        background-color: white;
    }

    .table-services tr:nth-child(odd) {
        background-color: #D9D9D9;
    }

    /* scroll bar */
    .table-services::-webkit-scrollbar {
        width: 12px;
    }

    .table-services::-webkit-scrollbar-thumb {
        background-color: #949494;
        border-radius: 10px;
    }

    .table-services::-webkit-scrollbar-track {
        background-color: #E2E2E2;
    }


	.floating-button, .suspend-button {
        position: fixed;
        z-index: 10;
    }

    .suspend-button {
        left: 65px;
        bottom: 40px;
    }

    .floating-button {
        right: 65px;
        bottom: 40px;
    }

    .suspend-button img, .floating-button img {
        margin-left: 10px;
        width: 24px;
        height: auto;
    }

    .suspend-button, .floating-button {
        background-color: #DC564E;
        color: white;
        cursor: pointer;
        font-size: 1.1em;
        padding: 12px 20px;
        width: 15em;
        height: auto;
        border-radius: 10px;
        border: none;
        cursor: pointer;
        /*font-weight: bold;*/
        align-items: center;
        justify-content: center;
        display: flex;
    }

    .right-arrow {
        width: 18px;
        margin-left: 10px;
        transform: rotate(180deg); 
        pointer-events: none; 
    }

    .btns {
        display: flex;
        flex-direction: column;
        flex-grow: 1;
        align-items: center;
    }
    
    .service-btn {
        background-color: #DC564E;
        color: white;
        padding: 8px 12px;
        cursor: pointer;
        font-size: 1.2vw;
        padding: 12px 20px;
		width: 300px; /* auto? é suposto adaptar ao texo?*/
		height: auto;
        border-radius: 1vmin;
        border: none;
        cursor: pointer;
        /*font-weight: bold;*/
        align-items: center;
        justify-content: center;
        display: flex;
    }

    .banner-shadow {
        background: linear-gradient(to right, rgba(125,125,125,0.3) 5%, rgba(204,204,204,0.6) 64%, rgba(204,204,204,0.6) 94%, rgba(255,255,255,0.7) 100%);
    }

    .page-shadow {
        min-height: calc(100vh - 30px); /* para começar em cima do footer  | */ 
        background: linear-gradient(to top, rgba(31,24,24,0.6) 0%, transparent 65%);
    }

    .status-message {
        font-size: 1.25em;
        margin-left: 2.4vw;
    }

    hr {
        border: none;
        height: 1px;
        background-color: rgb(194, 181, 181);
        margin: 40px 60px;
    }

    .separator-phone {
        display: none;
        height: 1px;
        width: 100%;
        background-color: black; 
        margin: 1em 0;
    }

    .red-text{
        color: var(--color-red);
   }

    @media (max-width: 700px) {

        .page{
            font-size: 0.8em;
        }

        .header {
            padding: 0 20px;
        }

        .info {
            display: flex;
            flex-direction: column;
            align-items: self-start;
        }

        .right {
            margin-bottom: 10px;
        }

        .estado {
            padding: 10px 0px 0px 15px;
            text-align: left;
        }

        .rectangle {
            width: 50px;
            height: auto;
        }

        .details {
            flex-direction: column;
            align-items: flex-start;
            padding: 20px 20px 0px 20px;
            
        }

        .veiculo, .cliente-title {
            padding: 0px 0px 15px 0px;
            font-size: 2.5em;
        }

        .car-info, .client-info {
            padding: 0px 0px 10px 0px;
            font-size: 1.7em;
        }

        .car-details{
            margin: 0px;
        }

        .tipo-motor {
            padding: 0px;
        }
        
        .motor {
            flex-direction: row;
            justify-content: space-between;
            padding: 0;
        }

        .motor img{
            width: 70px;
            height: auto;
        }

        .suspend-button, .floating-button{
            width: fit-content;
            font-size: 1em;
            padding: 10px 15px;
        }

        .suspend-button {
            left: 25px;
            bottom: 40px;
        }

        .floating-button {
            right: 25px;
            bottom: 40px;
        }

        .separator-phone {
            display: block;
        }
        
        .car-details .car-info:last-child {
            padding-bottom: 0;
        }

        hr {
            margin: 0;
        }

        .status {
            padding: 20px;
        }

        .status-message{
            margin: 0px;
        }

        .table-header{
            font-size: 4vw;
            margin-top: 20px;
        }

        .clock {
            display: none
        }

        .table-container {
            margin: 15px
        }
        .table-services {
            padding-right: 5px;
        }
    }

    

</style>


<!--
<template>
    <h1>Servico especifico page</h1>
    <iframe src="https://www.youtube.com/embed/dQw4w9WgXcQ?"></iframe>
</template>

<script>
export default {
    inheritAttrs: false,
}
</script>

<style scoped>
iframe {
    height:90vh;
    width: 100%;
}
</style>
-->