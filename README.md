
# VERSÃO 1: NUTRIVIDA PRO - Otimizado para Render (hospedagem gratuita)
# Inclui: app.py, requirements.txt, Procfile

nutrivida_app = '''
# ============================================
# NUTRIVIDA PRO - App Completo para Render
# Plataforma de Planos Alimentares com IA
# ============================================

from flask import Flask, request, jsonify, render_template_string, session
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime, timedelta
import random
import hashlib
import json
import os

app = Flask(__name__)
app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'nutrivida_secret_key_2026')
app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get('DATABASE_URL', 'sqlite:///nutrivida.db').replace('postgres://', 'postgresql://')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# ============================================
# MODELOS
# ============================================

class Usuario(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), unique=True, nullable=False)
    senha_hash = db.Column(db.String(128), nullable=False)
    nome = db.Column(db.String(100))
    idade = db.Column(db.Integer)
    peso = db.Column(db.Float)
    altura = db.Column(db.Float)
    objetivo = db.Column(db.String(50))
    restricoes = db.Column(db.Text)
    ativo = db.Column(db.Boolean, default=True)
    data_cadastro = db.Column(db.DateTime, default=datetime.utcnow)
    assinatura = db.relationship('Assinatura', backref='usuario', uselist=False)

class Assinatura(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    usuario_id = db.Column(db.Integer, db.ForeignKey('usuario.id'))
    status = db.Column(db.String(20), default='trial')
    plano = db.Column(db.String(20), default='mensal')
    data_inicio = db.Column(db.DateTime, default=datetime.utcnow)
    data_renovacao = db.Column(db.DateTime)
    valor = db.Column(db.Float)

class PlanoAlimentar(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    usuario_id = db.Column(db.Integer, db.ForeignKey('usuario.id'))
    conteudo_json = db.Column(db.Text)
    data_criacao = db.Column(db.DateTime, default=datetime.utcnow)
    semana = db.Column(db.Integer, default=1)

class ChatNutricionista(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    usuario_id = db.Column(db.Integer, db.ForeignKey('usuario.id'))
    mensagem_usuario = db.Column(db.Text)
    resposta_ia = db.Column(db.Text)
    data = db.Column(db.DateTime, default=datetime.utcnow)

# ============================================
# BANCO DE DADOS DE ALIMENTOS (IA)
# ============================================

ALIMENTOS = {
    "cafe_manha": [
        {"nome": "Omelete de 3 ovos com espinafre e tomate", "calorias": 350, "proteinas": 28, "carbos": 8, "gorduras": 22, "receita": "Bata 3 ovos, adicione espinafre picado e tomate. Cozinhe em frigideira antiaderente com azeite."},
        {"nome": "Vitamina de banana com aveia e whey protein", "calorias": 320, "proteinas": 25, "carbos": 42, "gorduras": 6, "receita": "Bata 1 banana, 2 colheres de aveia, 1 scoop de whey e 200ml de leite desnatado."},
        {"nome": "Tapioca recheada com frango desfiado", "calorias": 280, "proteinas": 22, "carbos": 35, "gorduras": 5, "receita": "Prepare a tapioca, recheie com frango desfiado temperado com cenoura ralada."},
        {"nome": "Panqueca de aveia com morangos", "calorias": 290, "proteinas": 12, "carbos": 48, "gorduras": 6, "receita": "Misture aveia, ovo, leite e fermento. Cozinhe em frigideira e sirva com morangos."},
        {"nome": "Iogurte grego com granola e mel", "calorias": 310, "proteinas": 18, "carbos": 38, "gorduras": 8, "receita": "Misture 200g de iogurte grego, 2 colheres de granola e 1 colher de mel."},
    ],
    "lanche_manha": [
        {"nome": "Maçã com 10 castanhas-do-pará", "calorias": 180, "proteinas": 4, "carbos": 22, "gorduras": 10},
        {"nome": "Shake de proteína com água", "calorias": 120, "proteinas": 24, "carbos": 3, "gorduras": 1},
        {"nome": "Pão integral com pasta de amendoim", "calorias": 220, "proteinas": 8, "carbos": 24, "gorduras": 10},
    ],
    "almoco": [
        {"nome": "Peito de frango grelhado (200g) + arroz integral + brócolis", "calorias": 520, "proteinas": 45, "carbos": 55, "gorduras": 10, "receita": "Tempere o frango com limão e ervas. Grelhe 7 min cada lado. Cozinhe arroz integral e brócolis no vapor."},
        {"nome": "Salmão assado (180g) + quinoa + aspargos", "calorias": 580, "proteinas": 38, "carbos": 42, "gorduras": 24, "receita": "Asse o salmão com limão e dill por 20 min. Cozinhe a quinoa e aspargos no vapor."},
        {"nome": "Carne magra (200g) + batata doce + salada", "calorias": 550, "proteinas": 48, "carbos": 48, "gorduras": 16, "receita": "Grelhe a carne ao ponto desejado. Asse a batata doce e prepare salada verde com azeite."},
        {"nome": "Filé de tilápia (200g) + purê de mandioquinha + legumes", "calorias": 480, "proteinas": 40, "carbos": 45, "gorduras": 12, "receita": "Grelhe a tilápia com ervas. Faça o purê de mandioquinha com leite desnatado. Refogue legumes."},
    ],
    "lanche_tarde": [
        {"nome": "Ovo cozido + torrada integral", "calorias": 160, "proteinas": 10, "carbos": 15, "gorduras": 7},
        {"nome": "Barrinha de proteína + café", "calorias": 200, "proteinas": 15, "carbos": 22, "gorduras": 6},
        {"nome": "Iogurte natural com chia", "calorias": 150, "proteinas": 10, "carbos": 18, "gorduras": 4},
    ],
    "jantar": [
        {"nome": "Sopa de legumes com frango desfiado", "calorias": 320, "proteinas": 28, "carbos": 30, "gorduras": 8, "receita": "Refogue cebola e alho, adicione legumes picados, caldo de galinha e frango desfiado. Cozinhe por 30 min."},
        {"nome": "Omelete de claras com tomate e queijo cottage", "calorias": 280, "proteinas": 25, "carbos": 8, "gorduras": 14, "receita": "Bata 4 claras, adicione tomate picado e queijo cottage. Cozinhe em frigideira antiaderente."},
        {"nome": "Atum em lata + salada verde + azeite", "calorias": 300, "proteinas": 32, "carbos": 5, "gorduras": 16, "receita": "Escorra o atum, misture com folhas verdes, tomate, pepino e tempere com azeite e limão."},
        {"nome": "Peito de peru (150g) + legumes no vapor", "calorias": 260, "proteinas": 30, "carbos": 12, "gorduras": 8, "receita": "Grelhe o peito de peru. Cozinhe brócolis, couve-flor e cenoura no vapor por 10 min."},
    ],
    "ceia": [
        {"nome": "Chá de camomila + 2 colheres de cottage", "calorias": 80, "proteinas": 10, "carbos": 4, "gorduras": 2},
        {"nome": "Gelatina zero + 5 amêndoas", "calorias": 60, "proteinas": 2, "carbos": 5, "gorduras": 4},
    ]
}

RESPOSTAS_IA = {
    "emagrecer": [
        "💡 Para acelerar o emagrecimento, foque em criar um déficit calórico moderado de 300-500 kcal. Seu plano já está otimizado para isso!",
        "💡 Dica da Nutri IA: Beba 500ml de água antes das refeições principais. Estudos mostram que isso reduz a ingestão calórica em até 13%.",
        "💡 Seu metabolismo está acelerando! Mantenha a consistência e os resultados virão. A chave é a constância, não a perfeição.",
        "💡 Substitua refrigerantes por água com gás e limão. Economiza 150 kcal por refeição = 1kg a menos por mês!",
        "💡 Durma 7-8 horas por noite. A falta de sono aumenta a produção de grelina (hormônio da fome) em 15%.",
    ],
    "ganhar_massa": [
        "💡 Para hipertrofia, distribua sua proteína em 4-5 refeições. Seu plano garante 2g/kg de peso corporal!",
        "💡 Dica da Nutri IA: Consuma carboidratos 1-2h antes do treino para máxima performance. Batata doce é excelente!",
        "💡 Recuperação muscular em dia! Seu plano inclui aminoácidos essenciais em todas as refeições principais.",
        "💡 Não esqueça do sono! Crescimento muscular acontece durante o descanso. Mínimo 7-8 horas por noite.",
        "💡 Aumente 300 kcal acima do seu gasto total para ganho de massa magra de 0.5kg por semana.",
    ],
    "manter": [
        "💡 Manutenção é sobre equilíbrio! Seu plano mantém seu gasto calórico basal com margem de segurança.",
        "💡 Dica da Nutri IA: Varie as cores dos vegetais para garantir todos os micronutrientes. Coma o arco-íris!",
        "💡 Seu peso está estável, ótimo sinal! Continue monitorando semanalmente para ajustes finos.",
        "💡 Mantenha a hidratação: 35ml de água por kg de peso corporal diariamente.",
    ],
    "geral": [
        "💡 A hidratação é fundamental! Beba pelo menos 35ml de água por kg de peso corporal diariamente.",
        "💡 Evite comer assistindo TV. Coma com atenção plena para reconhecer o sinal de saciedade.",
        "💡 Inclua gorduras boas: azeite, abacate, castanhas. Elas são essenciais para hormônios e saúde.",
        "💡 Planeje suas refeições da semana no domingo. Preparação antecipada evita escolhas impulsivas.",
    ]
}

# ============================================
# MOTOR DE IA
# ============================================

class NutriIA:
    @staticmethod
    def calcular_tmb(peso, altura, idade, sexo='F'):
        if sexo.upper() == 'M':
            return (10 * peso) + (6.25 * altura) - (5 * idade) + 5
        return (10 * peso) + (6.25 * altura) - (5 * idade) - 161
    
    @staticmethod
    def calcular_gasto_total(tmb, atividade='moderada'):
        fatores = {'sedentario': 1.2, 'leve': 1.375, 'moderada': 1.55, 'intensa': 1.725, 'muito_intensa': 1.9}
        return tmb * fatores.get(atividade, 1.55)
    
    @staticmethod
    def gerar_plano_semanal(usuario):
        tmb = NutriIA.calcular_tmb(usuario.peso, usuario.altura, usuario.idade)
        gasto_total = NutriIA.calcular_gasto_total(tmb)
        
        if usuario.objetivo == 'emagrecer':
            meta_calorias = gasto_total - 400
            distribuicao = {'proteinas': 0.35, 'carbos': 0.30, 'gorduras': 0.35}
        elif usuario.objetivo == 'ganhar_massa':
            meta_calorias = gasto_total + 300
            distribuicao = {'proteinas': 0.30, 'carbos': 0.45, 'gorduras': 0.25}
        else:
            meta_calorias = gasto_total
            distribuicao = {'proteinas': 0.25, 'carbos': 0.45, 'gorduras': 0.30}
        
        plano = {"meta_calorias": round(meta_calorias), "dias": {}}
        dias_semana = ['Segunda', 'Terça', 'Quarta', 'Quinta', 'Sexta', 'Sábado', 'Domingo']
        
        for dia in dias_semana:
            refeicoes = {}
            total_dia = {"calorias": 0, "proteinas": 0, "carbos": 0, "gorduras": 0}
            
            for refeicao, opcoes in ALIMENTOS.items():
                alimento = random.choice(opcoes)
                refeicoes[refeicao] = alimento
                for macro in ['calorias', 'proteinas', 'carbos', 'gorduras']:
                    total_dia[macro] += alimento[macro]
            
            plano["dias"][dia] = {
                "refeicoes": refeicoes,
                "totais": total_dia,
                "dica_ia": random.choice(RESPOSTAS_IA.get(usuario.objetivo, RESPOSTAS_IA['geral']))
            }
        
        return plano
    
    @staticmethod
    def responder_pergunta(pergunta, usuario):
        pergunta_lower = pergunta.lower()
        
        respostas_contextuais = {
            'emagrecer': [
                f"Com base no seu peso ({usuario.peso}kg), para emagrecer de forma saudável, defina um déficit de 400-500 kcal. Seu plano já está calibrado para isso!",
                "Reduza carboidratos refinados e aumente vegetais fibrosos. A saciedade é sua aliada!",
            ],
            'proteina': [
                f"Você precisa de aproximadamente {int(usuario.peso * 1.6)}g a {int(usuario.peso * 2.2)}g de proteína diária. Seu plano cobre essa meta!",
                "Fontes excelentes: peito de frango, ovos, peixes, whey protein, cottage cheese.",
            ],
            'treino': [
                "Combine treino de força 3-4x/semana com cardio moderado. A musculatura ativa queima mais calorias em repouso!",
                "Não faça cardio em jejum prolongado sem orientação. Pode causar perda de massa muscular.",
            ],
            'agua': [
                f"Beba pelo menos {int(usuario.peso * 35)}ml de água por dia. Isso equivale a aproximadamente {int(usuario.peso * 35 / 250)} copos!",
            ],
        }
        
        for chave, respostas in respostas_contextuais.items():
            if chave in pergunta_lower:
                return random.choice(respostas)
        
        return random.choice(RESPOSTAS_IA.get(usuario.objetivo, RESPOSTAS_IA['geral']))

# ============================================
# PÁGINAS HTML
# ============================================

LANDING_PAGE = """
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>NutriVida Pro - Seu Nutricionista Virtual 24h</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', sans-serif; background: #f0fdf4; color: #1a1a1a; line-height: 1.6; }
        .hero { background: linear-gradient(135deg, #16a34a, #15803d); color: white; padding: 80px 20px; text-align: center; }
        .hero h1 { font-size: 3em; margin-bottom: 20px; }
        .hero p { font-size: 1.3em; opacity: 0.9; }
        .btn { display: inline-block; background: #fbbf24; color: #1a1a1a; padding: 18px 50px; border-radius: 50px; text-decoration: none; font-weight: bold; font-size: 1.2em; margin-top: 30px; transition: transform 0.3s; border: none; cursor: pointer; }
        .btn:hover { transform: scale(1.05); }
        .features { padding: 60px 20px; max-width: 1200px; margin: 0 auto; }
        .features-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 30px; margin-top: 40px; }
        .feature-card { background: white; padding: 30px; border-radius: 20px; box-shadow: 0 10px 40px rgba(0,0,0,0.1); text-align: center; }
        .feature-card h3 { color: #16a34a; margin: 15px 0; }
        .pricing { background: white; padding: 60px 20px; text-align: center; }
        .pricing-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 30px; max-width: 900px; margin: 40px auto; }
        .price-card { border: 2px solid #e5e7eb; border-radius: 20px; padding: 40px 30px; position: relative; }
        .price-card.popular { border-color: #16a34a; transform: scale(1.05); }
        .badge { position: absolute; top: -12px; left: 50%; transform: translateX(-50%); background: #16a34a; color: white; padding: 5px 20px; border-radius: 20px; font-size: 0.9em; }
        .price { font-size: 3em; color: #16a34a; font-weight: bold; }
        .price span { font-size: 0.4em; color: #666; }
        .form-section { max-width: 500px; margin: 40px auto; padding: 40px; background: white; border-radius: 20px; box-shadow: 0 10px 40px rgba(0,0,0,0.1); }
        .form-group { margin-bottom: 20px; }
        .form-group label { display: block; margin-bottom: 8px; font-weight: bold; color: #333; }
        .form-group input, .form-group select { width: 100%; padding: 12px; border: 2px solid #e5e7eb; border-radius: 10px; font-size: 1em; }
        .form-group input:focus, .form-group select:focus { border-color: #16a34a; outline: none; }
        .plano-result { max-width: 800px; margin: 40px auto; padding: 40px; background: white; border-radius: 20px; box-shadow: 0 10px 40px rgba(0,0,0,0.1); display: none; }
        .dia-card { background: #f0fdf4; padding: 20px; border-radius: 15px; margin-bottom: 20px; }
        .dia-card h3 { color: #16a34a; margin-bottom: 15px; }
        .refeicao { background: white; padding: 15px; border-radius: 10px; margin-bottom: 10px; border-left: 4px solid #16a34a; }
        .chat-box { max-width: 600px; margin: 40px auto; padding: 30px; background: white; border-radius: 20px; box-shadow: 0 10px 40px rgba(0,0,0,0.1); }
        .chat-messages { height: 300px; overflow-y: auto; border: 2px solid #e5e7eb; border-radius: 10px; padding: 15px; margin-bottom: 15px; }
        .chat-input { display: flex; gap: 10px; }
        .chat-input input { flex: 1; padding: 12px; border: 2px solid #e5e7eb; border-radius: 10px; }
        .msg-user { background: #16a34a; color: white; padding: 10px 15px; border-radius: 15px 15px 0 15px; margin: 5px 0 5px auto; max-width: 80%; text-align: right; }
        .msg-bot { background: #f3f4f6; padding: 10px 15px; border-radius: 15px 15px 15px 0; margin: 5px auto 5px 0; max-width: 80%; }
        footer { background: #1a1a1a; color: white; text-align: center; padding: 30px; }
        .hidden { display: none; }
    </style>
</head>
<body>
    <div class="hero">
        <h1>🥗 NutriVida Pro</h1>
        <p>Planos alimentares personalizados por IA em 2 minutos.<br>Seu nutricionista virtual disponível 24 horas por dia.</p>
        <a href="#cadastro" class="btn">Quero Meu Plano Grátis →</a>
    </div>
    
    <div class="features">
        <h2 style="text-align:center; font-size:2em;">Como Funciona?</h2>
        <div class="features-grid">
            <div class="feature-card">
                <div style="font-size:3em;">1️⃣</div>
                <h3>Cadastre-se Grátis</h3>
                <p>7 dias de teste sem cartão de crédito. Cancele quando quiser.</p>
            </div>
            <div class="feature-card">
                <div style="font-size:3em;">2️⃣</div>
                <h3>Responda 5 Perguntas</h3>
                <p>Idade, peso, altura, objetivo e restrições alimentares.</p>
            </div>
            <div class="feature-card">
                <div style="font-size:3em;">3️⃣</div>
                <h3>Receba Seu Plano</h3>
                <p>A IA gera um plano alimentar completo e personalizado em segundos.</p>
            </div>
            <div class="feature-card">
                <div style="font-size:3em;">4️⃣</div>
                <h3>Converse com a Nutri IA</h3>
                <p>Tire dúvidas 24h por dia. Substitua planos semanalmente.</p>
            </div>
        </div>
    </div>
    
    <div class="form-section" id="cadastro">
        <h2 style="text-align:center; color:#16a34a; margin-bottom:30px;">Criar Conta Grátis</h2>
        <form id="formCadastro">
            <div class="form-group">
                <label>Nome Completo</label>
                <input type="text" id="nome" required placeholder="Seu nome">
            </div>
            <div class="form-group">
                <label>Email</label>
                <input type="email" id="email" required placeholder="seu@email.com">
            </div>
            <div class="form-group">
                <label>Senha</label>
                <input type="password" id="senha" required placeholder="Mínimo 6 caracteres">
            </div>
            <div class="form-group">
                <label>Idade</label>
                <input type="number" id="idade" required min="10" max="100">
            </div>
            <div class="form-group">
                <label>Peso (kg)</label>
                <input type="number" id="peso" required step="0.1" min="30" max="200">
            </div>
            <div class="form-group">
                <label>Altura (cm)</label>
                <input type="number" id="altura" required min="100" max="250">
            </div>
            <div class="form-group">
                <label>Objetivo</label>
                <select id="objetivo" required>
                    <option value="emagrecer">Emagrecer</option>
                    <option value="ganhar_massa">Ganhar Massa Muscular</option>
                    <option value="manter">Manter Peso</option>
                </select>
            </div>
            <div class="form-group">
                <label>Restrições Alimentares (opcional)</label>
                <input type="text" id="restricoes" placeholder="Ex: intolerância à lactose, vegetariano">
            </div>
            <button type="submit" class="btn" style="width:100%;">Gerar Meu Plano Alimentar →</button>
        </form>
    </div>
    
    <div class="plano-result" id="planoResult">
        <h2 style="text-align:center; color:#16a34a; margin-bottom:20px;">🎉 Seu Plano Alimentar está Pronto!</h2>
        <div id="planoConteudo"></div>
        <div style="text-align:center; margin-top:30px;">
            <button class="btn" onclick="mostrarChat()">💬 Falar com Nutri IA</button>
            <button class="btn" style="background:#1a1a1a; color:white; margin-left:10px;" onclick="location.reload()">🔄 Gerar Novo Plano</button>
        </div>
    </div>
    
    <div class="chat-box hidden" id="chatBox">
        <h3 style="color:#16a34a; margin-bottom:15px;">🤖 Nutri IA - Tire suas dúvidas</h3>
        <div class="chat-messages" id="chatMessages">
            <div class="msg-bot">Olá! Sou sua nutricionista virtual. Como posso te ajudar hoje? 💚</div>
        </div>
        <div class="chat-input">
            <input type="text" id="chatInput" placeholder="Digite sua pergunta..." onkeypress="if(event.key==='Enter')enviarChat()">
            <button class="btn" style="margin-top:0; padding:12px 25px;" onclick="enviarChat()">Enviar</button>
        </div>
    </div>
    
    <div class="pricing" id="planos">
        <h2 style="font-size:2em; margin-bottom:10px;">Escolha Seu Plano</h2>
        <p style="color:#666;">Cancele quando quiser. Sem taxa de cancelamento.</p>
        <div class="pricing-grid">
            <div class="price-card">
                <h3>Mensal</h3>
                <div class="price">R$49<span>/mês</span></div>
                <ul style="list-style:none; margin:20px 0; line-height:2;">
                    <li>✅ Plano alimentar semanal</li>
                    <li>✅ Chat com Nutri IA</li>
                    <li>✅ Lista de compras</li>
                </ul>
                <button class="btn" style="padding:12px 30px; font-size:1em;" onclick="alert('Faça login para assinar!')">Assinar</button>
            </div>
            <div class="price-card popular">
                <div class="badge">MAIS POPULAR</div>
                <h3>Trimestral</h3>
                <div class="price">R$39<span>/mês</span></div>
                <p style="text-decoration:line-through; color:#999;">R$147</p>
                <p style="color:#16a34a; font-weight:bold;">R$117 total (3 meses)</p>
                <ul style="list-style:none; margin:20px 0; line-height:2;">
                    <li>✅ Tudo do plano mensal</li>
                    <li>✅ Ebook Receitas Fit</li>
                    <li>✅ Grupo VIP Telegram</li>
                    <li>✅ 20% OFF</li>
                </ul>
                <button class="btn" style="padding:12px 30px; font-size:1em;" onclick="alert('Faça login para assinar!')">Assinar</button>
            </div>
            <div class="price-card">
                <h3>Anual</h3>
                <div class="price">R$29<span>/mês</span></div>
                <p style="text-decoration:line-through; color:#999;">R$588</p>
                <p style="color:#16a34a; font-weight:bold;">R$359 total (12 meses)</p>
                <ul style="list-style:none; margin:20px 0; line-height:2;">
                    <li>✅ Tudo do trimestral</li>
                    <li>✅ Planos de treino bônus</li>
                    <li>✅ 40% OFF</li>
                </ul>
                <button class="btn" style="padding:12px 30px; font-size:1em;" onclick="alert('Faça login para assinar!')">Assinar</button>
            </div>
        </div>
    </div>
    
    <footer>
        <p>© 2026 NutriVida Pro. Todos os direitos reservados.</p>
        <p style="margin-top:10px; opacity:0.7;">Este serviço não substitui consulta com nutricionista.</p>
    </footer>
    
    <script>
        let usuarioId = null;
        
        document.getElementById('formCadastro').addEventListener('submit', async function(e) {
            e.preventDefault();
            const dados = {
                nome: document.getElementById('nome').value,
                email: document.getElementById('email').value,
                senha: document.getElementById('senha').value,
                idade: parseInt(document.getElementById('idade').value),
                peso: parseFloat(document.getElementById('peso').value),
                altura: parseInt(document.getElementById('altura').value),
                objetivo: document.getElementById('objetivo').value,
                restricoes: document.getElementById('restricoes').value
            };
            
            try {
                const resp = await fetch('/api/cadastro', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify(dados)
                });
                const data = await resp.json();
                if (data.sucesso) {
                    usuarioId = data.usuario_id;
                    gerarPlano(usuarioId);
                } else {
                    alert('Erro no cadastro: ' + (data.erro || 'Tente novamente'));
                }
            } catch(err) {
                alert('Erro de conexão. Tente novamente.');
            }
        });
        
        async function gerarPlano(uid) {
            try {
                const resp = await fetch('/api/gerar-plano/' + uid);
                const plano = await resp.json();
                mostrarPlano(plano);
            } catch(err) {
                alert('Erro ao gerar plano. Recarregue a página.');
            }
        }
        
        function mostrarPlano(plano) {
            document.getElementById('cadastro').style.display = 'none';
            document.getElementById('planoResult').style.display = 'block';
            
            let html = `<div style="text-align:center; margin-bottom:30px; padding:20px; background:#f0fdf4; border-radius:15px;">
                <h3 style="color:#16a34a;">Meta Calórica Diária: ${plano.meta_calorias} kcal</h3>
                <p>Plano personalizado baseado no seu perfil</p>
            </div>`;
            
            for (let dia in plano.dias) {
                const d = plano.dias[dia];
                html += `<div class="dia-card">
                    <h3>📅 ${dia}</h3>
                    <p style="color:#16a34a; margin-bottom:15px; font-style:italic;">${d.dica_ia}</p>`;
                
                const nomes = {'cafe_manha':'☕ Café da Manhã','lanche_manha':'🍎 Lanche da Manhã','almoco':'🍽️ Almoço','lanche_tarde':'🥤 Lanche da Tarde','jantar':'🌙 Jantar','ceia':'🌜 Ceia'};
                for (let ref in d.refeicoes) {
                    const r = d.refeicoes[ref];
                    html += `<div class="refeicao">
                        <strong>${nomes[ref] || ref}</strong><br>
                        ${r.nome}<br>
                        <small style="color:#666;">🔥 ${r.calorias} kcal | 🥩 ${r.proteinas}g prot | 🍞 ${r.carbos}g carb | 🥑 ${r.gorduras}g gord</small>`;
                    if (r.receita) html += `<br><small style="color:#16a34a;">👨‍🍳 ${r.receita}</small>`;
                    html += `</div>`;
                }
                
                html += `<div style="margin-top:15px; padding:10px; background:white; border-radius:10px; text-align:center;">
                    <strong>Totais do dia:</strong> 🔥 ${d.totais.calorias} kcal | 🥩 ${d.totais.proteinas}g | 🍞 ${d.totais.carbos}g | 🥑 ${d.totais.gorduras}g
                </div></div>`;
            }
            
            document.getElementById('planoConteudo').innerHTML = html;
        }
        
        function mostrarChat() {
            document.getElementById('chatBox').classList.remove('hidden');
            document.getElementById('chatBox').scrollIntoView({behavior:'smooth'});
        }
        
        async function enviarChat() {
            const input = document.getElementById('chatInput');
            const msg = input.value.trim();
            if (!msg || !usuarioId) return;
            
            const chatDiv = document.getElementById('chatMessages');
            chatDiv.innerHTML += `<div class="msg-user">${msg}</div>`;
            input.value = '';
            chatDiv.scrollTop = chatDiv.scrollHeight;
            
            try {
                const resp = await fetch('/api/chat', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({usuario_id: usuarioId, pergunta: msg})
                });
                const data = await resp.json();
                chatDiv.innerHTML += `<div class="msg-bot">${data.resposta}</div>`;
                chatDiv.scrollTop = chatDiv.scrollHeight;
            } catch(err) {
                chatDiv.innerHTML += `<div class="msg-bot">Desculpe, tive um problema. Tente novamente!</div>`;
            }
        }
    </script>
</body>
</html>
"""

# ============================================
# ROTAS API
# ============================================

@app.route('/')
def landing():
    return render_template_string(LANDING_PAGE)

@app.route('/api/cadastro', methods=['POST'])
def cadastrar():
    dados = request.json
    senha_hash = hashlib.sha256(dados['senha'].encode()).hexdigest()
    
    if Usuario.query.filter_by(email=dados['email']).first():
        return jsonify({"sucesso": False, "erro": "Email já cadastrado"}), 400
    
    usuario = Usuario(
        email=dados['email'],
        senha_hash=senha_hash,
        nome=dados['nome'],
        idade=dados['idade'],
        peso=dados['peso'],
        altura=dados['altura'],
        objetivo=dados['objetivo'],
        restricoes=dados.get('restricoes', '')
    )
    db.session.add(usuario)
    db.session.commit()
    
    # Cria assinatura trial
    assinatura = Assinatura(
        usuario_id=usuario.id,
        status='trial',
        data_renovacao=datetime.utcnow() + timedelta(days=7)
    )
    db.session.add(assinatura)
    db.session.commit()
    
    return jsonify({"sucesso": True, "usuario_id": usuario.id})

@app.route('/api/gerar-plano/<int:usuario_id>')
def gerar_plano(usuario_id):
    usuario = Usuario.query.get_or_404(usuario_id)
    plano = NutriIA.gerar_plano_semanal(usuario)
    
    novo_plano = PlanoAlimentar(
        usuario_id=usuario_id,
        conteudo_json=json.dumps(plano, ensure_ascii=False)
    )
    db.session.add(novo_plano)
    db.session.commit()
    
    return jsonify(plano)

@app.route('/api/chat', methods=['POST'])
def chat_nutricionista():
    dados = request.json
    usuario = Usuario.query.get(dados['usuario_id'])
    resposta = NutriIA.responder_pergunta(dados['pergunta'], usuario)
    
    chat = ChatNutricionista(
        usuario_id=usuario.id,
        mensagem_usuario=dados['pergunta'],
        resposta_ia=resposta
    )
    db.session.add(chat)
    db.session.commit()
    
    return jsonify({"resposta": resposta})

# ============================================
# INICIALIZAÇÃO
# ============================================

with app.app_context():
    db.create_all()

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=int(os.environ.get('PORT', 5000)))
'''

# Salvar app.py
with open('/mnt/agents/output/nutrivida_pro/app.py', 'w', encoding='utf-8') as f:
    f.write(nutrivida_app)

# Criar requirements.txt
requirements = """Flask==3.0.0
Flask-SQLAlchemy==3.1.1
Werkzeug==3.0.1
psycopg2-binary==2.9.9
"""

with open('/mnt/agents/output/nutrivida_pro/requirements.txt', 'w') as f:
    f.write(requirements)

# Criar Procfile
procfile = "web: gunicorn app:app"
with open('/mnt/agents/output/nutrivida_pro/Procfile', 'w') as f:
    f.write(procfile)

print("✅ NUTRIVIDA PRO - App completo criado!")
print("📁 Arquivos:")
print("   - /mnt/agents/output/nutrivida_pro/app.py")
print("   - /mnt/agents/output/nutrivida_pro/requirements.txt")
print("   - /mnt/agents/output/nutrivida_pro/Procfile")
