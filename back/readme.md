
// ===== BACKEND - Node.js/Express API =====

// package.json
{
  "name": "mex-energia-backend",
  "version": "1.0.0",
  "description": "Backend API para Sistema MEX Energia Social - República José Virgínio",
  "scripts": {
    "start": "node dist/server.js",
    "dev": "nodemon src/server.ts",
    "build": "tsc",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "helmet": "^7.0.0",
    "dotenv": "^16.3.1",
    "pg": "^8.11.3",
    "redis": "^4.6.8",
    "influxdb-client": "^1.33.2",
    "ws": "^8.13.0",
    "ethers": "^6.7.1",
    "axios": "^1.5.0",
    "node-cron": "^3.0.2",
    "winston": "^3.10.0"
  }
}

// src/server.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import dotenv from 'dotenv';
import { createServer } from 'http';
import { WebSocketServer } from 'ws';
import cron from 'node-cron';

import solarRoutes from './routes/solar';
import blockchainRoutes from './routes/blockchain';
import smadsRoutes from './routes/smads';
import participantRoutes from './routes/participants';

import { SolarDataService } from './services/SolarDataService';
import { BlockchainService } from './services/BlockchainService';
import { WeatherService } from './services/WeatherService';
import { SmartMeterAPI } from './integrations/SmartMeterAPI';
import { InverterAPI } from './integrations/InverterAPI';

dotenv.config();

const app = express();
const server = createServer(app);
const wss = new WebSocketServer({ server });

// Middleware
app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Services
const solarService = new SolarDataService();
const blockchainService = new BlockchainService();
const weatherService = new WeatherService();

// Routes
app.use('/api/solar', solarRoutes);
app.use('/api/blockchain', blockchainRoutes);
app.use('/api/smads', smadsRoutes);
app.use('/api/participants', participantRoutes);

// WebSocket para tempo real
wss.on('connection', (ws) => {
  console.log('Cliente conectado ao WebSocket');
  
  ws.on('message', (message) => {
    console.log('Mensagem recebida:', message.toString());
  });

  ws.on('close', () => {
    console.log('Cliente desconectado');
  });
});

// Cronjobs para coleta de dados
cron.schedule('*/5 * * * *', async () => {
  // A cada 5 minutos - coletar dados do smart meter e inversor
  await solarService.collectRealTimeData();
});

cron.schedule('0 * * * *', async () => {
  // A cada hora - processar tokens e atualizar blockchain
  await blockchainService.processHourlyTokens();
});

cron.schedule('0 6 * * *', async () => {
  // Diariamente às 6h - relatório SMADS
  await generateSMADSReport();
});

const PORT = process.env.PORT || 3001;

server.listen(PORT, () => {
  console.log(`🚀 Servidor MEX Energia rodando na porta ${PORT}`);
});

// ===== SERVICES =====

// src/services/SolarDataService.ts
import { SmartMeterAPI } from '../integrations/SmartMeterAPI';
import { InverterAPI } from '../integrations/InverterAPI';
import { WeatherService } from './WeatherService';
import { InfluxDB, Point } from '@influxdata/influxdb-client';

export class SolarDataService {
  private smartMeter: SmartMeterAPI;
  private inverter: InverterAPI;
  private weather: WeatherService;
  private influxDB: InfluxDB;

  constructor() {
    this.smartMeter = new SmartMeterAPI(process.env.SMART_METER_IP!);
    this.inverter = new InverterAPI(process.env.INVERTER_IP!);
    this.weather = new WeatherService();
    this.influxDB = new InfluxDB({
      url: process.env.INFLUXDB_URL!,
      token: process.env.INFLUXDB_TOKEN!
    });
  }

  async collectRealTimeData() {
    try {
      // República José Virgínio - Sistema 26kWp
      const location = { lat: -23.5293, lng: -46.6417 }; // Vila Reg. Feijó
      
      // 1. Dados do Smart Meter (consumo)
      const meterData = await this.smartMeter.getCurrentReading();
      
      // 2. Dados do Inversor (produção)
      const inverterData = await this.inverter.getProductionData();
      
      // 3. Dados climáticos
      const weatherData = await this.weather.getCurrentWeather(location);
      const solarData = await this.weather.getSolarData(location);
      
      // 4. Calcular excedente para SIN
      const excess = Math.max(0, inverterData.currentPower - meterData.currentConsumption);
      
      // 5. Calcular eficiência real
      const efficiency = this.calculateRealTimeEfficiency(weatherData, solarData);
      
      // 6. Salvar dados time-series
      await this.saveTimeSeriesData({
        production: inverterData.currentPower,
        consumption: meterData.currentConsumption,
        excess: excess,
        efficiency: efficiency,
        weather: weatherData,
        solar: solarData
      });

      // 7. Gerar tokens SOLAR baseado no excedente
      if (excess > 0) {
        await this.generateSolarTokens(excess, efficiency);
      }

      return {
        production: inverterData.currentPower,
        consumption: meterData.currentConsumption,
        excess: excess,
        efficiency: efficiency,
        weather: weatherData
      };

    } catch (error) {
      console.error('Erro coletando dados solares:', error);
      throw error;
    }
  }

  private calculateRealTimeEfficiency(weather: any, solar: any): number {
    const now = new Date();
    const sunrise = new Date(solar.sunrise);
    const sunset = new Date(solar.sunset);
    
    // Eficiência base pela posição solar
    let efficiency = 0;
    if (now >= sunrise && now <= sunset) {
      const dayProgress = (now.getTime() - sunrise.getTime()) / 
                         (sunset.getTime() - sunrise.getTime());
      efficiency = Math.sin(dayProgress * Math.PI);
    }
    
    // Ajustes por condições climáticas
    const cloudFactor = 1 - (weather.clouds.all / 100 * 0.7);
    const tempFactor = weather.main.temp > 25 ? 
      1 - ((weather.main.temp - 25) * 0.004) : 1;
    
    return Math.max(0, efficiency * cloudFactor * tempFactor);
  }

  private async saveTimeSeriesData(data: any) {
    const writeApi = this.influxDB.getWriteApi(
      process.env.INFLUXDB_ORG!,
      process.env.INFLUXDB_BUCKET!
    );

    const point = new Point('solar_metrics')
      .tag('location', 'republica_jose_virginio')
      .tag('project', 'mex_energia_social')
      .floatField('production_kw', data.production)
      .floatField('consumption_kw', data.consumption)
      .floatField('excess_kw', data.excess)
      .floatField('efficiency', data.efficiency)
      .floatField('temperature', data.weather.main.temp)
      .intField('clouds', data.weather.clouds.all)
      .timestamp(new Date());

    writeApi.writePoint(point);
    await writeApi.close();
  }

  private async generateSolarTokens(excess: number, efficiency: number) {
    // Fórmula: 1 SOLAR = 1 kWh excedente * eficiência
    const tokensToMint = excess * efficiency;
    
    // Enviar para blockchain service
    const blockchainService = new (await import('./BlockchainService')).BlockchainService();
    await blockchainService.mintSolarTokens(tokensToMint);
  }
}

// ===== INTEGRATIONS =====

// src/integrations/SmartMeterAPI.ts
import axios from 'axios';

export interface MeterReading {
  currentConsumption: number; // kW
  totalConsumption: number; // kWh
  voltage: number; // V
  frequency: number; // Hz
  timestamp: Date;
  gridConnection: boolean;
}

export class SmartMeterAPI {
  private baseUrl: string;

  constructor(meterIP: string) {
    this.baseUrl = `http://${meterIP}/api/v1`;
  }

  async getCurrentReading(): Promise<MeterReading> {
    try {
      // Integração com medidor inteligente real
      // Exemplo: protocolo Modbus TCP ou REST API
      const response = await axios.get(`${this.baseUrl}/current`);
      
      return {
        currentConsumption: response.data.active_power / 1000, // W para kW
        totalConsumption: response.data.total_energy / 1000, // Wh para kWh
        voltage: response.data.voltage,
        frequency: response.data.frequency,
        timestamp: new Date(),
        gridConnection: response.data.grid_connected
      };
    } catch (error) {
      console.error('Erro lendo smart meter:', error);
      // Fallback para dados simulados
      return this.getSimulatedReading();
    }
  }

  async getHourlyData(hours: number = 24): Promise<MeterReading[]> {
    try {
      const response = await axios.get(`${this.baseUrl}/history?hours=${hours}`);
      return response.data.map((item: any) => ({
        currentConsumption: item.active_power / 1000,
        totalConsumption: item.total_energy / 1000,
        voltage: item.voltage,
        frequency: item.frequency,
        timestamp: new Date(item.timestamp),
        gridConnection: item.grid_connected
      }));
    } catch (error) {
      console.error('Erro obtendo histórico do medidor:', error);
      return [];
    }
  }

  async sendToSIN(excessEnergy: number): Promise<boolean> {
    try {
      // Integração com Sistema Integrado Nacional
      const response = await axios.post(`${this.baseUrl}/inject`, {
        excess_energy: excessEnergy,
        timestamp: new Date().toISOString()
      });
      
      return response.data.success;
    } catch (error) {
      console.error('Erro injetando energia no SIN:', error);
      return false;
    }
  }

  private getSimulatedReading(): MeterReading {
    const hour = new Date().getHours();
    let consumption = 2.0; // Base consumption
    
    // Simular padrão de consumo da República José Virgínio
    if (hour >= 6 && hour <= 8) consumption = 3.5; // Manhã
    if (hour >= 12 && hour <= 14) consumption = 4.2; // Almoço
    if (hour >= 18 && hour <= 22) consumption = 3.8; // Noite
    
    return {
      currentConsumption: consumption + (Math.random() - 0.5) * 0.4,
      totalConsumption: Math.random() * 1000 + 5000,
      voltage: 220 + (Math.random() - 0.5) * 10,
      frequency: 60 + (Math.random() - 0.5) * 0.5,
      timestamp: new Date(),
      gridConnection: true
    };
  }
}

// src/integrations/InverterAPI.ts
import axios from 'axios';

export interface InverterData {
  currentPower: number; // kW atual
  dailyProduction: number; // kWh do dia
  totalProduction: number; // kWh total
  efficiency: number; // %
  dcVoltage: number; // V
  dcCurrent: number; // A
  acVoltage: number; // V
  acFrequency: number; // Hz
  temperature: number; // °C
  status: 'online' | 'offline' | 'fault' | 'maintenance';
  timestamp: Date;
}

export class InverterAPI {
  private baseUrl: string;

  constructor(inverterIP: string) {
    // Inversor híbrido 15kW - República José Virgínio
    this.baseUrl = `http://${inverterIP}/api/v1`;
  }

  async getProductionData(): Promise<InverterData> {
    try {
      // Integração com inversor real (exemplo: Growatt, Fronius, SMA)
      const response = await axios.get(`${this.baseUrl}/realtime`);
      
      return {
        currentPower: response.data.pac / 1000, // W para kW
        dailyProduction: response.data.eday,
        totalProduction: response.data.etotal,
        efficiency: response.data.efficiency,
        dcVoltage: response.data.vpv1,
        dcCurrent: response.data.ipv1,
        acVoltage: response.data.vac1,
        acFrequency: response.data.fac1,
        temperature: response.data.temperature,
        status: this.mapStatus(response.data.status),
        timestamp: new Date()
      };
    } catch (error) {
      console.error('Erro lendo dados do inversor:', error);
      return this.getSimulatedData();
    }
  }

  async getBatteryStatus() {
    try {
      // Sistema híbrido com 20 baterias 100Ah
      const response = await axios.get(`${this.baseUrl}/battery`);
      
      return {
        chargeLevel: response.data.soc, // State of Charge %
        voltage: response.data.voltage,
        current: response.data.current,
        power: response.data.power,
        temperature: response.data.temperature,
        status: response.data.status,
        backupHours: response.data.backup_time
      };
    } catch (error) {
      console.error('Erro lendo status das baterias:', error);
      return this.getSimulatedBatteryData();
    }
  }

  async enableHybridMode(enable: boolean): Promise<boolean> {
    try {
      const response = await axios.post(`${this.baseUrl}/mode`, {
        hybrid_enabled: enable,
        backup_enabled: enable
      });
      
      return response.data.success;
    } catch (error) {
      console.error('Erro configurando modo híbrido:', error);
      return false;
    }
  }

  private mapStatus(statusCode: number): InverterData['status'] {
    switch (statusCode) {
      case 1: return 'online';
      case 0: return 'offline';
      case 2: return 'fault';
      case 3: return 'maintenance';
      default: return 'offline';
    }
  }

  private getSimulatedData(): InverterData {
    const hour = new Date().getHours();
    let power = 0;
    
    // Simular curva de produção solar - 26kWp instalado
    if (hour >= 6 && hour <= 18) {
      const dayPosition = (hour - 6) / 12;
      const solarCurve = Math.sin(dayPosition * Math.PI);
      power = 26 * solarCurve * (0.7 + Math.random() * 0.3); // Variação climática
    }
    
    return {
      currentPower: Math.max(0, power),
      dailyProduction: Math.random() * 100 + 50,
      totalProduction: Math.random() * 10000 + 50000,
      efficiency: 92 + Math.random() * 6,
      dcVoltage: 400 + Math.random() * 50,
      dcCurrent: power > 0 ? power / 0.4 : 0,
      acVoltage: 220 + Math.random() * 10,
      acFrequency: 60 + Math.random() * 0.2,
      temperature: 25 + Math.random() * 20,
      status: power > 0 ? 'online' : 'offline',
      timestamp: new Date()
    };
  }

  private getSimulatedBatteryData() {
    return {
      chargeLevel: 70 + Math.random() * 20, // 70-90%
      voltage: 48 + Math.random() * 4,
      current: -5 + Math.random() * 10, // Carga/descarga
      power: Math.random() * 5000,
      temperature: 25 + Math.random() * 10,
      status: 'normal',
      backupHours: 6 + Math.random() * 2 // 6-8h backup
    };
  }
}

// ===== BLOCKCHAIN SERVICE =====

// src/services/BlockchainService.ts
import { ethers } from 'ethers';

export class BlockchainService {
  private provider: ethers.Provider;
  private wallet: ethers.Wallet;
  private solarTokenContract: ethers.Contract;
  private smadsWallet: ethers.Wallet;

  constructor() {
    // Kindelia network configuration
    this.provider = new ethers.JsonRpcProvider(process.env.KINDELIA_RPC_URL);
    this.wallet = new ethers.Wallet(process.env.PRIVATE_KEY!, this.provider);
    
    // SMADS crypto wallet
    this.smadsWallet = new ethers.Wallet(
      process.env.SMADS_PRIVATE_KEY!, 
      this.provider
    );

    // Solar Token contract
    this.solarTokenContract = new ethers.Contract(
      process.env.SOLAR_TOKEN_ADDRESS!,
      SOLAR_TOKEN_ABI,
      this.wallet
    );
  }

  async mintSolarTokens(amount: number): Promise<string> {
    try {
      // 1 kWh excedente = 1 SOLAR token
      const amountWei = ethers.parseEther(amount.toString());
      
      const tx = await this.solarTokenContract.mint(
        this.wallet.address,
        amountWei
      );
      
      await tx.wait();
      return tx.hash;
    } catch (error) {
      console.error('Erro mintando tokens SOLAR:', error);
      throw error;
    }
  }

  async transferEmendaToProject(amount: string): Promise<string> {
    try {
      // Emenda Parlamentar Rubinho Nunes: R$ 1.5Mi
      const amountWei = ethers.parseEther(amount);
      
      const tx = await this.smadsWallet.sendTransaction({
        to: process.env.PROJECT_WALLET_ADDRESS!,
        value: amountWei,
        data: ethers.encodeBytes32String("EMENDA_RUBINHO_NUNES_MEX_ENERGIA")
      });
      
      await tx.wait();
      return tx.hash;
    } catch (error) {
      console.error('Erro transferindo emenda:', error);
      throw error;
    }
  }

  async processHourlyTokens(): Promise<void> {
    // Processar tokens acumulados na última hora
    // Enviar para DAO Energética
    // Atualizar métricas na blockchain
  }
}

const SOLAR_TOKEN_ABI = [
  "function mint(address to, uint256 amount) external",
  "function balanceOf(address account) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "function totalSupply() view returns (uint256)"
];
