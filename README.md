<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Moto Táxi Pro</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        :root { --primary: #2ecc71; --driver: #3498db; --danger: #e74c3c; --dark: #2c3e50; }
        
        * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }

        body { 
            font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; 
            background-color: #f8f9fa; 
            margin: 0; 
            padding: 0;
            display: flex;
            flex-direction: column;
            height: 100vh;
            overflow: hidden;
        }

        /* Topo Fixo */
        .header-nav {
            background: var(--dark);
            color: white;
            padding: 12px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            font-size: 13px;
            z-index: 1000;
        }

        /* Container Principal */
        .app-content {
            flex: 1;
            position: relative;
            display: flex;
            flex-direction: column;
        }

        #map { flex: 1; width: 100%; z-index: 1; }

        /* Painéis Flutuantes (Estilo Uber) */
        .floating-card {
            background: white;
            padding: 15px;
            border-radius: 20px 20px 0 0;
            box-shadow: 0 -2px 10px rgba(0,0,0,0.1);
            z-index: 1000;
            position: relative;
        }

        .painel { display: none; }
        .ativo { display: block; }

        /* Estilos de Texto e Inputs */
        h3 { margin: 0 0 10px 0; font-size: 18px; color: var(--dark); }
        .route-info { background: #f1f3f5; padding: 10px; border-radius: 10px; margin-bottom: 10px; }
        .price { font-size: 24px; font-weight: bold; color: var(--primary); margin: 10px 0; }
        
        input { 
            width: 100%; border: none; background: transparent; 
            font-size: 16px; color: #555; padding: 5px 0;
        }

        /* Botões Mobile */
        .btn {
            width: 100%;
            padding: 16px;
            border: none;
            border-radius: 12px;
            font-size: 16px;
            font-weight: bold;
            color: white;
            cursor: pointer;
            margin-top: 5px;
        }
        .btn-green { background: var(--primary); }
        .btn-blue { background: var(--driver); }
        .btn-admin { background: #f39c12; }

        /* Card do Motorista */
        .driver-profile {
            display: flex;
            align-items: center;
            gap: 15px;
            margin-bottom: 15px;
            text-align: left;
        }
        .photo-frame {
            position: relative;
            width: 60px;
            height: 60px;
        }
        .photo-frame img {
            width: 100%; height: 100%; border-radius: 50%; object-fit: cover;
            border: 3px solid var(--driver);
        }

        /* Lista Admin */
        .admin-list { max-height: 300px; overflow-y: auto; text-align: left; }
        .motorista-row {
            display: flex; justify-content: space-between; align-items: center;
            padding: 10px; border-bottom: 1px solid #eee;
        }
    </style>
</head>
<body>

    <div class="header-nav">
        <strong>MOTO TÁXI PRO</strong>
        <div>
            <span onclick="mudarTela('painel-passageiro')">Início</span> | 
            <span onclick="abrirLoginMotorista()">Piloto</span> | 
            <span onclick="abrirAdmin()" style="color:#f39c12">Admin</span>
        </div>
    </div>

    <div class="app-content">
        <div id="map"></div>

        <div id="painel-passageiro" class="floating-card painel ativo">
            <h3>Para onde vamos?</h3>
            <div class="route-info">
                <small style="color: #888; font-weight: bold;">DESTINO</small>
                <input type="text" id="destinoInput" placeholder="Toque no mapa para marcar" readonly>
            </div>
            <div class="price" id="estimativa-preco">R$ 0,00</div>
            <button class="btn btn-green" onclick="enviarChamada()">CONFIRMAR MOTOCICLETA</button>
        </div>

        <div id="painel-motorista" class="floating-card painel">
            <div class="driver-profile">
                <div class="photo-frame" onclick="mudarMinhaFoto()">
                    <img id="m-foto" src="" alt="foto">
                    <div style="position:absolute; bottom:0; right:0; background:white; border-radius:50%; padding:2px; font-size:10px;">✏️</div>
                </div>
                <div>
                    <strong id="m-nome">Piloto</strong><br>
                    <span style="color:var(--primary); font-weight:bold">Saldo: <span id="m-saldo">R$ 0,00</span></span>
                </div>
            </div>

            <div id="chamada-box" style="display:none; background:#ebf5fb; padding:15px; border-radius:15px; border:2px solid var(--driver)">
                <small style="color:var(--driver)">NOVA CORRIDA DISPONÍVEL</small>
                <div id="m-preco-p" class="price" style="margin:5px 0">R$ 0,00</div>
                <button class="btn btn-blue" onclick="aceitarCorrida()">ACEITAR E ABRIR GPS</button>
            </div>
            <p id="msg-espera" style="color:#999; margin:20px 0;">Aguardando chamadas na sua área...</p>
            <button class="btn" style="background:#eee; color:#333;" onclick="location.reload()">SAIR</button>
        </div>

        <div id="painel-admin" class="floating-card painel">
            <h3 style="color:#f39c12">Gerir Pilotos</h3>
            <div id="lista-motoristas" class="admin-list"></div>
            <button class="btn" style="background:#333" onclick="mudarTela('painel-passageiro')">FECHAR</button>
        </div>
    </div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
    import { getDatabase, ref, set, onValue, update } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-database.js";

    // Configuração do Banco de Dados
    const firebaseConfig = { databaseURL: "https://moto-taxi-pro-default-rtdb.firebaseio.com" };
    const app = initializeApp(firebaseConfig);
    const db = getDatabase(app);

    let map, markerP, markerD, latP, lonP, latD, lonD, motoristaLogado = null;

    window.mudarTela = (id) => {
        document.querySelectorAll('.painel').forEach(p => p.classList.remove('ativo'));
        document.getElementById(id).classList.add('ativo');
    };

    // Inicializar Mapa Móvel
    navigator.geolocation.getCurrentPosition(pos => {
        latP = pos.coords.latitude; lonP = pos.coords.longitude;
        map = L.map('map', {zoomControl: false}).setView([latP, lonP], 16);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
        
        markerP = L.circleMarker([latP, lonP], {radius: 8, color: '#3498db', fillOpacity: 1}).addTo(map);

        map.on('click', e => {
            latD = e.latlng.lat; lonD = e.latlng.lng;
            if (markerD) map.removeLayer(markerD);
            markerD = L.marker([latD, lonD]).addTo(map);
            
            let dist = (map.distance([latP, lonP], [latD, lonD]) / 1000);
            let preco = dist <= 3 ? 5.00 : 5.00 + (dist-3)*1.50;
            document.getElementById('estimativa-preco').innerText = `R$ ${preco.toFixed(2)}`;
            document.getElementById('destinoInput').value = "Destino Selecionado";
            // Vibrar ao marcar (se suportado)
            if(navigator.vibrate) navigator.vibrate(50);
        });
    });

    // Funções do Passageiro
    window.enviarChamada = () => {
        if(!latD) return alert("Toque no mapa para escolher o destino!");
        set(ref(db, 'chamada_ativa'), {
            lat: latD, lon: lonD,
            valor: document.getElementById('estimativa-preco').innerText,
            status: "pendente",
            timestamp: Date.now()
        }).then(() => alert("Procurando pilotos próximos..."));
    };

    // Funções do Motorista
    window.abrirLoginMotorista = () => {
        const id = prompt("ID do Piloto:");
        if(!id) return;
        onValue(ref(db, 'motoristas/' + id), snap => {
            const m = snap.val();
            if(m) {
                motoristaLogado = id;
                mudarTela('painel-motorista');
                document.getElementById('m-nome').innerText = m.nome;
                document.getElementById('m-saldo').innerText = `R$ ${m.saldo.toFixed(2)}`;
                document.getElementById('m-foto').src = m.foto || "https://i.pravatar.cc/150?u="+id;
            } else {
                set(ref(db, 'motoristas/' + id), { nome: "Piloto "+id, saldo: 0, foto: "" });
            }
        });

        onValue(ref(db, 'chamada_ativa'), snap => {
            const c = snap.val();
            if(c && c.status === "pendente") {
                document.getElementById('chamada-box').style.display = 'block';
                document.getElementById('msg-espera').style.display = 'none';
                document.getElementById('m-preco-p').innerText = c.valor;
                window.lastChamada = c;
                if(navigator.vibrate) navigator.vibrate([200, 100, 200]);
            } else {
                document.getElementById('chamada-box').style.display = 'none';
                document.getElementById('msg-espera').style.display = 'block';
            }
        });
    };

    window.mudarMinhaFoto = () => {
        const url = prompt("Link da Foto (URL):");
        if(url && motoristaLogado) update(ref(db, 'motoristas/'+motoristaLogado), { foto: url });
    };

    window.aceitarCorrida = () => {
        update(ref(db, 'chamada_ativa'), { status: "aceita" });
        window.open(`https://www.google.com/maps/dir/?api=1&destination=${window.lastChamada.lat},${window.lastChamada.lon}&travelmode=motorcycle`);
    };

    // Administração
    window.abrirAdmin = () => {
        if(prompt("Senha Admin:") === "admin123") {
            mudarTela('painel-admin');
            onValue(ref(db, 'motoristas'), snap => {
                const container = document.getElementById('lista-motoristas');
                container.innerHTML = "";
                const data = snap.val();
                for(let id in data) {
                    container.innerHTML += `
                        <div class="motorista-row">
                            <span>${data[id].nome}<br><small>R$ ${data[id].saldo.toFixed(2)}</small></span>
                            <div>
                                <button class="btn-sm" onclick="ajustar('${id}', 1)">+1</button>
                                <button class="btn-sm" onclick="ajustar('${id}', -1)">-1</button>
                            </div>
                        </div>`;
                }
            });
        }
    };

    window.ajustar = (id, v) => {
        onValue(ref(db, `motoristas/${id}/saldo`), s => {
            update(ref(db, `motoristas/${id}`), { saldo: (s.val() || 0) + v });
        }, {onlyOnce: true});
    };

</script>
</body>
</html>
