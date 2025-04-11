# License Validator System

Sistema de validação de licenças com PostgreSQL.

## Instalação

```bash
npm install
```

## Variáveis de Ambiente

Crie um arquivo `.env` com:

```env
DATABASE_URL=sua_url_postgres
SECRET_KEY=sua_chave_secreta
ADMIN_KEY=sua_chave_admin
PORT=3000
NODE_ENV=production
```

## Exemplos de Uso

### 1. Criar Nova Licença (Admin)

```javascript
const axios = require('axios');

async function criarLicenca(adminKey, licenseKey, validityDays = 365, checksPerDay = 5) {
    try {
        const response = await axios.post('https://seu-servidor/create-license', {
            adminKey,
            licenseKey,
            validityDays,
            checksPerDay
        });
        return response.data;
    } catch (error) {
        throw new Error(error.response?.data?.message || error.message);
    }
}

// Exemplo de uso
criarLicenca('sua-chave-admin', 'NOVA-LICENCA-123', 365, 5)
    .then(console.log)
    .catch(console.error);
```

### 2. Validar Licença (Cliente)

```javascript
const axios = require('axios');
const crypto = require('crypto-js');

class LicenseValidator {
    constructor(licenseKey, secretKey, serverUrl) {
        this.licenseKey = licenseKey;
        this.secretKey = secretKey;
        this.serverUrl = serverUrl;
    }

    generateHash(timestamp) {
        return crypto.SHA256(
            `${this.licenseKey}-${timestamp}-${this.secretKey}`
        ).toString();
    }

    async validate() {
        try {
            const timestamp = new Date().toISOString();
            const clientHash = this.generateHash(timestamp);

            const response = await axios.post(`${this.serverUrl}/check-license`, {
                key: this.licenseKey,
                timestamp: timestamp,
                clientHash: clientHash
            });

            return {
                isValid: response.data.status === 'active',
                validUntil: response.data.validUntil,
                checksRemaining: response.data.checksRemaining,
                ...response.data
            };
        } catch (error) {
            if (error.response?.status === 403) {
                return { isValid: false, error: 'Licença inválida ou expirada' };
            }
            throw new Error(error.response?.data?.message || error.message);
        }
    }
}

// Exemplo de uso em uma aplicação
async function iniciarAplicacao() {
    const validator = new LicenseValidator(
        'SUA-LICENCA-123',      // Chave da licença
        'sua-chave-secreta',    // Chave secreta
        'https://seu-servidor'  // URL do servidor
    );

    try {
        const result = await validator.validate();
        
        if (result.isValid) {
            console.log('✅ Licença válida!');
            console.log(`📅 Válida até: ${new Date(result.validUntil).toLocaleDateString()}`);
            console.log(`🔄 Verificações restantes hoje: ${result.checksRemaining}`);
            // Inicie sua aplicação aqui
        } else {
            console.error('❌ Licença inválida:', result.error);
            process.exit(1);
        }
    } catch (error) {
        console.error('❌ Erro ao validar licença:', error);
        process.exit(1);
    }
}

// Exemplo com verificação periódica
class LicenseManager {
    constructor(licenseKey, secretKey, serverUrl, checkInterval = 30) {
        this.validator = new LicenseValidator(licenseKey, secretKey, serverUrl);
        this.checkInterval = checkInterval * 60 * 1000; // Converte minutos para ms
        this.isValid = false;
    }

    async startChecking() {
        const check = async () => {
            try {
                const result = await this.validator.validate();
                this.isValid = result.isValid;
                
                if (!this.isValid) {
                    console.error('❌ Licença expirada ou inválida');
                    process.exit(1);
                }
            } catch (error) {
                console.error('❌ Erro na verificação:', error);
            }
        };

        // Primeira verificação
        await check();
        
        // Verificações periódicas
        setInterval(check, this.checkInterval);
    }
}

// Uso do LicenseManager
const manager = new LicenseManager(
    'SUA-LICENCA-123',
    'sua-chave-secreta',
    'https://seu-servidor',
    30 // Verificar a cada 30 minutos
);

manager.startChecking().catch(console.error);
```

### 3. Exemplo em Python

```python
import hashlib
import requests
from datetime import datetime
import json

class LicenseValidator:
    def __init__(self, license_key: str, secret_key: str, server_url: str):
        self.license_key = license_key
        self.secret_key = secret_key
        self.server_url = server_url

    def generate_hash(self, timestamp: str) -> str:
        message = f"{self.license_key}-{timestamp}-{self.secret_key}"
        return hashlib.sha256(message.encode()).hexdigest()

    def validate(self) -> dict:
        timestamp = datetime.utcnow().isoformat()
        client_hash = self.generate_hash(timestamp)

        try:
            response = requests.post(
                f"{self.server_url}/check-license",
                json={
                    "key": self.license_key,
                    "timestamp": timestamp,
                    "clientHash": client_hash
                },
                headers={"Content-Type": "application/json"}
            )
            
            response.raise_for_status()
            data = response.json()
            
            return {
                "isValid": data["status"] == "active",
                "validUntil": data["validUntil"],
                "checksRemaining": data["checksRemaining"]
            }
        except requests.exceptions.RequestException as e:
            if e.response is not None and e.response.status_code == 403:
                return {"isValid": False, "error": "Licença inválida ou expirada"}
            raise Exception(f"Erro ao validar licença: {str(e)}")

# Exemplo de uso
validator = LicenseValidator(
    "SUA-LICENCA-123",
    "sua-chave-secreta",
    "https://seu-servidor"
)

try:
    result = validator.validate()
    if result["isValid"]:
        print("✅ Licença válida!")
        print(f"📅 Válida até: {result['validUntil']}")
        print(f"🔄 Verificações restantes: {result['checksRemaining']}")
    else:
        print(f"❌ Erro: {result.get('error', 'Licença inválida')}")
except Exception as e:
    print(f"❌ Erro: {str(e)}")
```

## Códigos de Erro

- `400`: Parâmetros inválidos ou faltando
- `403`: Licença inválida/expirada ou acesso não autorizado
- `429`: Limite diário de verificações excedido
- `500`: Erro interno do servidor

## Segurança

- Use HTTPS para todas as comunicações
- Armazene as chaves de forma segura (variáveis de ambiente)
- Implemente rate limiting no cliente
- Mantenha o `SECRET_KEY` em segurança
- Não compartilhe o `ADMIN_KEY`

## Notas

- O timestamp não pode ser mais antigo que 5 minutos
- Cada licença tem um limite diário de verificações
- As licenças podem estar: active, inactive, ou expired
- O servidor usa PostgreSQL para armazenamento persistente
