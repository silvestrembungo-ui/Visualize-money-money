# Visualize-money-money
App de remoneração por visualização
# Visualize Money

Visualize Money é um protótipo de plataforma de vídeos remunerados (Play-and-Earn). Este repositório contém uma scaffold mínima (backend em Node + Express + SQLite via Sequelize) com modelos, rotas e páginas públicas de Política de Privacidade e Termos de Uso. Serve como ponto de partida para desenvolvimento e implantação.

Principais características implementadas no scaffold:
- Cadastro / Login (email + número telefónico de Angola + senha forte).
- Recuperação de senha por email ou número (fluxo básico).
- Áreas de Produtor e Visualizador (rotas API básicas).
- Upload de vídeo / incorporação (YouTube / Facebook / TikTok) com definição de orçamento (cada visualização válida custa 8 Kz).
- Fluxo de envio de comprovativo (referência: entidade 00579, referência: 461864531).
- Validação de visualização: mínimo 2 minutos, máximo 3 visualizações seguidas, espera de 2h se guardar.
- Ganho do visualizador: 2 Kz por visualização válida.
- Saques: 1x por mês, retenção de 20%, PIN para saque.
- Política de convite: 1% do saque do convidado durante 4 meses (esqueleto).
- Painéis (endpoints) para produtores e visualizadores (estatísticas básicas).
- Política básica de prevenção a fraude (stubs).

Aviso importante sobre credenciais de admin
- O app inicializa um utilizador admin com as credenciais fornecidas:
  - Email: silvestre.m.bifica@gmail.com
  - Senha: 921777754marla*
- Essas credenciais são inseridas no arquivo `.env` durante o seed (veja `.env.example`).
- Em produção: NÃO use credenciais em texto limpo; rode migrações, mude a senha e use secrets seguros.

Como usar (local)
1. Clone o repositório e cole os ficheiros neste repositório no GitHub.
2. Copie `.env.example` para `.env` e ajuste conforme necessário.
3. Instale dependências:
   - npm install
4. Execute migração/seed:
   - node src/seed.js
5. Inicie o servidor:
   - npm start
6. API disponível por padrão em: http://localhost:3000

Estrutura principal
- src/
  - app.js - servidor Express
  - models.js - modelos Sequelize (User, Video, View, Transaction, Invite)
  - seed.js - cria o admin e dados iniciais
  - routes/
    - auth.js
    - producer.js
    - viewer.js
  - utils/
    - fraud.js - stubs de prevenção de fraudes
- public/
  - privacy-policy.md
  - terms.md

Notas finais
- Este scaffold é um ponto de partida. Para produção: adicione HTTPS, validação completa de inputs, testes, serviço de armazenamento (S3), CDN, processamento de vídeo, worker para contagem de visualizações, monitorização e limites de taxa.
- A implementação aqui é intencionalmente simples para facilitar revisão e personalização.

Contato do projeto
- visualizemoney.sb@gmail.com
{
  "name": "visualize-money",
  "version": "0.1.0",
  "description": "Plataforma de vídeos remunerados - protótipo",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "seed": "node src/seed.js"
  },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "body-parser": "^1.20.2",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.1",
    "sequelize": "^6.32.1",
    "sqlite3": "^5.1.6",
    "uuid": "^9.0.0"
  }
}
# Copie para .env e ajuste antes de usar
PORT=3000
JWT_SECRET=trocar_por_segredo_forte
DATABASE_FILE=./database.sqlite

# Admin inicial (apenas para seed; altere em produção)
ADMIN_EMAIL=silvestre.m.bifica@gmail.com
ADMIN_PASSWORD=921777754marla*
ADMIN_NAME=Admin Visualize Money

# Configurações de pagamento (exemplo)
PAYMENT_ENTITY=00579
PAYMENT_REFERENCE=461864531

# Outros parâmetros do sistema
VIEW_COST_KZ=8
VIEWER_REWARD_KZ=2
// Servidor Express básico com rotas de autenticação, produtor e visualizador.
require('dotenv').config();
const express = require('express');
const bodyParser = require('body-parser');
const { initModels } = require('./models');
const authRoutes = require('./routes/auth');
const producerRoutes = require('./routes/producer');
const viewerRoutes = require('./routes/viewer');

const app = express();
app.use(bodyParser.json());

(async () => {
  const { sequelize } = await initModels();
  // Sincroniza DB (use migrations em produção)
  await sequelize.sync();

  // Rotas
  app.use('/api/auth', authRoutes);
  app.use('/api/producer', producerRoutes);
  app.use('/api/viewer', viewerRoutes);

  // Páginas públicas
  app.use('/privacy-policy', (req, res) => res.sendFile(require('path').resolve('./public/privacy-policy.md')));
  app.use('/terms', (req, res) => res.sendFile(require('path').resolve('./public/terms.md')));

  const PORT = process.env.PORT || 3000;
  app.listen(PORT, () => console.log(`Visualize Money API rodando em http://localhost:${PORT}`));
})();
// Sequelize models: User, Video, View, Transaction, Invite
const { Sequelize, DataTypes } = require('sequelize');
const path = require('path');

let sequelize;

async function initModels() {
  const dbFile = process.env.DATABASE_FILE || path.resolve(__dirname, '../database.sqlite');
  sequelize = new Sequelize({
    dialect: 'sqlite',
    storage: dbFile,
    logging: false
  });

  const User = sequelize.define('User', {
    id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
    name: DataTypes.STRING,
    email: { type: DataTypes.STRING, unique: true },
    phone: DataTypes.STRING, // validar +244 Angola no frontend
    passwordHash: DataTypes.STRING,
    role: { type: DataTypes.STRING, defaultValue: 'viewer' }, // viewer | producer | admin
    balanceKz: { type: DataTypes.FLOAT, defaultValue: 0 },
    withdrawPin: DataTypes.STRING,
    invitedBy: DataTypes.UUID
  });

  const Video = sequelize.define('Video', {
    id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
    title: DataTypes.STRING,
    description: DataTypes.TEXT,
    ownerId: DataTypes.UUID,
    sourceType: DataTypes.STRING, // upload | youtube | facebook | tiktok
    sourceUrl: DataTypes.STRING,
    budgetKz: { type: DataTypes.FLOAT, defaultValue: 0 },
    spentKz: { type: DataTypes.FLOAT, defaultValue: 0 },
    approved: { type: DataTypes.BOOLEAN, defaultValue: false },
    playCount: { type: DataTypes.INTEGER, defaultValue: 0 }
  });

  const View = sequelize.define('View', {
    id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
    videoId: DataTypes.UUID,
    userId: DataTypes.UUID,
    durationSeconds: DataTypes.INTEGER,
    valid: { type: DataTypes.BOOLEAN, defaultValue: false },
    createdAt: { type: DataTypes.DATE, defaultValue: Sequelize.NOW }
  }, { timestamps: true, updatedAt: false });

  const Transaction = sequelize.define('Transaction', {
    id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
    userId: DataTypes.UUID,
    type: DataTypes.STRING, // deposit, spend, reward, withdraw
    amountKz: DataTypes.FLOAT,
    meta: DataTypes.JSON
  });

  const Invite = sequelize.define('Invite', {
    id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
    inviterId: DataTypes.UUID,
    inviteeId: DataTypes.UUID,
    startDate: DataTypes.DATE,
    monthsLeft: DataTypes.INTEGER
  });

  // Associações simples
  User.hasMany(Video, { foreignKey: 'ownerId' });
  Video.belongsTo(User, { foreignKey: 'ownerId' });

  Video.hasMany(View, { foreignKey: 'videoId' });
  View.belongsTo(Video, { foreignKey: 'videoId' });

  User.hasMany(View, { foreignKey: 'userId' });
  View.belongsTo(User, { foreignKey: 'userId' });

  User.hasMany(Transaction, { foreignKey: 'userId' });

  return { sequelize, User, Video, View, Transaction, Invite };
}

module.exports = { initModels };
// Seed inicial: cria admin com credenciais do .env
require('dotenv').config();
const bcrypt = require('bcrypt');
const { initModels } = require('./models');

(async () => {
  const { sequelize, User } = await initModels();
  await sequelize.sync();

  const adminEmail = process.env.ADMIN_EMAIL;
  const adminPassword = process.env.ADMIN_PASSWORD;
  const adminName = process.env.ADMIN_NAME || 'Admin';

  if (!adminEmail || !adminPassword) {
    console.error('Defina ADMIN_EMAIL e ADMIN_PASSWORD no .env');
    process.exit(1);
  }

  const existing = await User.findOne({ where: { email: adminEmail } });
  if (existing) {
    console.log('Admin já existe:', adminEmail);
    process.exit(0);
  }

  const passwordHash = await bcrypt.hash(adminPassword, 10);
  await User.create({
    name: adminName,
    email: adminEmail,
    passwordHash,
    role: 'admin',
    balanceKz: 0
  });

  console.log('Admin criado:', adminEmail);
  process.exit(0);
})();
// Rotas de autenticação: register, login, recuperar senha (stubs)
const express = require('express');
const router = express.Router();
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const { initModels } = require('../models');
const { Op } = require('sequelize');

const JWT_SECRET = process.env.JWT_SECRET || 'trocar_por_segredo_forte';

let modelsCache;
async function models() {
  if (!modelsCache) modelsCache = await initModels();
  return modelsCache;
}

// Register: email, phone (Angola), password
router.post('/register', async (req, res) => {
  const { name, email, phone, password, inviteCode } = req.body;
  if (!email || !password) return res.status(400).json({ error: 'email e password obrigatórios' });
  // validar phone para Angola (+244 ou 9xx...), deixar simples aqui
  const { User } = await models();
  const exists = await User.findOne({ where: { [Op.or]: [{ email }, { phone }] } });
  if (exists) return res.status(400).json({ error: 'Usuário já existe' });
  const passwordHash = await bcrypt.hash(password, 10);
  const user = await User.create({ name, email, phone, passwordHash, role: 'viewer' });
  // Invites: criar lógica se inviteCode presente
  return res.json({ ok: true, userId: user.id });
});

// Login
router.post('/login', async (req, res) => {
  const { emailOrPhone, password } = req.body;
  const { User } = await models();
  const user = await User.findOne({ where: { [Op.or]: [{ email: emailOrPhone }, { phone: emailOrPhone }] } });
  if (!user) return res.status(401).json({ error: 'Credenciais inválidas' });
  const match = await bcrypt.compare(password, user.passwordHash);
  if (!match) return res.status(401).json({ error: 'Credenciais inválidas' });
  const token = jwt.sign({ userId: user.id, role: user.role }, JWT_SECRET, { expiresIn: '30d' });
  res.json({ token, role: user.role });
});

// Recuperar senha (por email ou telefone) - stub: retorna link temporário
router.post('/recover', async (req, res) => {
  const { emailOrPhone } = req.body;
  // Em produção, enviar email ou SMS com token. Aqui apenas responde OK.
  res.json({ ok: true, message: 'Se a conta existir, enviámos instruções para recuperar por email ou SMS.' });
});

module.exports = router;
// Rotas para produtores: criar vídeo (upload/incorporar), enviar comprovativo, dashboard
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const { initModels } = require('../models');
const { Op } = require('sequelize');
const fraud = require('../utils/fraud');

const JWT_SECRET = process.env.JWT_SECRET || 'trocar_por_segredo_forte';
const VIEW_COST = Number(process.env.VIEW_COST_KZ || 8);

let modelsCache;
async function models() {
  if (!modelsCache) modelsCache = await initModels();
  return modelsCache;
}

function authMiddleware(req, res, next) {
  const auth = req.headers.authorization;
  if (!auth) return res.status(401).json({ error: 'Sem token' });
  const token = auth.replace('Bearer ', '');
  try {
    const data = jwt.verify(token, JWT_SECRET);
    req.user = data;
    next();
  } catch (e) { return res.status(401).json({ error: 'Token inválido' }); }
}

// Criar vídeo (incorporar link ou apontar file) com orçamento
router.post('/videos', authMiddleware, async (req, res) => {
  const { title, description, sourceType, sourceUrl, budgetKz } = req.body;
  const { Video, User } = await models();
  const owner = await User.findByPk(req.user.userId);
  if (!owner) return res.status(404).json({ error: 'Usuário não encontrado' });
  // Apenas produtores podem enviar (promover role)
  // Para simplicidade, permitimos todos criarem e ficam pendentes de aprovação.
  const video = await Video.create({
    title, description, sourceType, sourceUrl, ownerId: owner.id, budgetKz: Number(budgetKz || 0), approved: false
  });
  // In production: gerar instruções de pagamento com entidade/ref
  res.json({ ok: true, videoId: video.id, payment: { entity: process.env.PAYMENT_ENTITY, reference: process.env.PAYMENT_REFERENCE } });
});

// Enviar comprovativo (stubs) e aguardar revisão
router.post('/videos/:id/proof', authMiddleware, async (req, res) => {
  const { id } = req.params;
  const { proofUrl } = req.body;
  // Em produção, salvar prova e notificar admin para revisar.
  res.json({ ok: true, message: 'Comprovativo recebido, aguardando revisão.' });
});

// Dashboard do produtor: visualizações, gasto, relatórios básicos
router.get('/dashboard', authMiddleware, async (req, res) => {
  const { Video, View } = await models();
  // Apenas videos do usuário
  const videos = await Video.findAll({ where: { ownerId: req.user.userId } });
  // agregações simples
  const dashboard = await Promise.all(videos.map(async v => {
    const views = await View.count({ where: { videoId: v.id, valid: true } });
    return { videoId: v.id, title: v.title, views, budgetKz: v.budgetKz, spentKz: v.spentKz };
  }));
  res.json({ ok: true, videos: dashboard });
});

module.exports = router;
// Stubs simples para prevenção de fraudes.
// Em produção: usar heurísticas mais avançadas, análise de IP, user-agent, comportamento, rate limiting, análise por worker.
module.exports = {
  isLikelyFraud(req) {
    // Exemplo: bloquear se user-agent ausente ou IP privado (muito simplista)
    const ua = req.headers['user-agent'] || '';
    if (!ua) return true;
    // Poderia verificar headers, cookies, padrões de tempo, etc.
    return false;
  }
};
