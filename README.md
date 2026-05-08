<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Moto Táxi Pro - Gestão Total</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        :root { --primary: #28a745; --driver-blue: #1967d2; --danger: #dc3545; --admin-gold: #f39c12; }
        body { font-family: 'Segoe UI', sans-serif; background-color: #f4f4f9; margin: 0; display: flex; justify-content: center; min-height: 100vh; }
        .container { background: #fff; padding: 1.5rem; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); text-align: center; max-width: 450px; width: 95%; position: relative; margin-top: 20px; height: fit-content; }
        
        #map { height: 200px; width: 100%; border-radius: 8px; margin-bottom: 10px; border: 1px solid #ddd; }
        .route-info { text-align: left; background: #f9f9f9; padding: 12px; border-radius: 8px; margin-bottom: 10px; border-left: 5px solid var(--primary); }
        .price-tag { font-size: 1.5rem; color: var(--primary); font-weight: bold; margin: 10px 0; }
        .btn-main { background-color: var(--primary); color: white; border: none; padding: 15px; border-radius: 8px; cursor: pointer; font-size: 1rem; font-weight: bold; width: 100%; margin-bottom: 10px; }
        
        /* Modos */
        .painel { display: none; }
        .ativo { display: block; }

        /* Painel Admin */
        .admin-item { display: flex; align-items: center; justify-content: space-between; background: #eee; padding: 10px; border-radius: 8px; margin-bottom: 5px; font-size: 0.9rem; }
        .btn-sm { padding: 5px 10px; font-size: 0.7rem; cursor: pointer; }
        
        .driver-header { display: flex; align-items: center; gap: 10px; text-align: left; margin-bottom: 15px; background: #eef2ff; padding: 10px; border-radius: 8px; position: relative; }
        .driver-photo { width: 50px; height: 50px; border-radius: 50%; object-fit: cover; background: #ccc; border: 2px solid var(--driver-blue); }
        .nav-links { position: absolute; top: 5px; right: 10px; font-size: 0.65rem; color: #aaa; }
    </style>
</head>
<body>

<div class="container">
    <div class="nav-links">
        <span onclick="mudarTela('painel-passageiro')" style="cursor:pointer">Passageiro</span> | 
        <span onclick="abrirLoginMotorista()" style="cursor:pointer">Piloto</span> | 
        <span onclick="abrirAdmin()" style="cursor:pointer; color:var(--admin-gold)">Admin</span>
    </div>

    <div id="painel-passageiro" class="painel ativo">
        <h3 style="margin-top:20px">Pedir Moto Táxi</h3>
        <div id="map"></div>
        <div class="route-info">
            <small>Destino:</small>
            <input type="text" id="destinoInput" style="width:100%; border:none; background:transparent; font-weight:bold" placeholder="Toque no mapa" readonly>
        </div>
        <div id="estimativa-preco" class="price-tag">R$ 0,00</div>
        <button class="btn-main" onclick="enviarChamada()">SOLICITAR AGORA</button>
    </div>

    <div id="painel-motorista" class="painel">
        <div class="driver-header">
            <img id="m-foto" class="driver-photo" src="" alt="foto" onclick="mudarMinhaFoto()">
            <div>
                <strong id="m-nome">Carregando...</strong><br>
                <small>Saldo: <span id="m-saldo" style="color:green; font-weight:bold">R$ 0,00</span></small>
            </div>
            <button class="btn-sm" style="position:absolute; right:10px" onclick="mudarMinhaFoto()">Foto</button>
        </div>
        <div id="area-chamada" style="display:none; background:#e8f0fe; padding:15px; border-radius:8px">
            <h4 style="margin:0; color:var(--driver-blue)">NOVA CORRIDA!</h4>
            <p id="m-dest-p" style="font-weight:bold; margin:10px 0"></p>
            <div id="m-preco-p" class="price-tag" style="margin:0"></div>
            <button class="btn-main" style="background:var(--driver-blue); margin-top:10px" onclick="aceitarCorrida()">ACEITAR E NAVEGAR</button>
        </div>
        <p id="esperando" style="color:#888; padding:20px">Aguardando pedidos...</p>
    </div>

    <div id="painel-admin" class="painel">
        <h3 style="color:var(--admin-gold)">Gestão de Pilotos</h3>
        <div id="lista-motoristas"></div>
        <button class="btn-main" style="background:#666; margin-top:10px" onclick="mudarTela('painel-passageiro')">VOLTAR</button>
    </div>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
    import { getDatabase, ref, set, onValue, update } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-database.js";

    const firebaseConfig = { databaseURL: "https://moto-taxi-pro-default-rtdb.firebaseio.com" };
    const app = initializeApp(firebaseConfig);
    const db = getDatabase(app);

    let map, markerP, markerD, latP, lonP, latD, lonD, motoristaLogado = null;

    // --- FUNÇÕES DE NAVEGAÇÃO ---
    window.mudarTela = (id) => {
        document.querySelectorAll('.painel').forEach(p => p.classList.remove('ativo'));
        document.getElementById(id).classList.add('ativo');
    };

    // --- LÓGICA PASSAGEIRO ---
    navigator.geolocation.getCurrentPosition(pos => {
        latP = pos.coords.latitude; lonP = pos.coords.longitude;
        map = L.map('map').setView([latP, lonP], 15);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
        markerP = L.marker([latP, lonP]).addTo(map);
        map.on('click', e => {
            latD = e.latlng.lat; lonD = e.latlng.lng;
            if (markerD) map.removeLayer(markerD);
            markerD = L.marker([latD, lonD]).addTo(map);
            let dist = (map.distance([latP, lonP], [latD, lonD]) / 1000);
            let preco = dist <= 3 ? 5.00 : 5.00 + (dist-3)*1.50;
            document.getElementById('estimativa-preco').innerText = `R$ ${preco.toFixed(2)}`;
            document.getElementById('destinoInput').value = "Destino marcado no mapa";
        });
    });

    window.enviarChamada = () => {
        if(!latD) return alert("Selecione o destino!");
        set(ref(db, 'chamada_ativa'), {
            destino: { lat: latD, lon: lonD, nome: "Corrida Solicitada" },
            valor: document.getElementById('estimativa-preco').innerText,
            status: "pendente"
        }).then(() => alert("Buscando pilotos..."));
    };

    // --- LÓGICA MOTORISTA ---
    window.abrirLoginMotorista = () => {
        const id = prompt("Seu ID de Piloto:");
        if(!id) return;
        onValue(ref(db, 'motoristas/' + id), snap => {
            const m = snap.val();
            if(m) {
                motoristaLogado = id;
                mudarTela('painel-motorista');
                document.getElementById('m-nome').innerText = m.nome;
                document.getElementById('m-saldo').innerText = `R$ ${m.saldo.toFixed(2)}`;
                document.getElementById('m-foto').src = m.foto || "https://via.placeholder.com/150";
            } else {
                set(ref(db, 'motoristas/' + id), { nome: "Piloto "+id, saldo: 0, foto: "" });
            }
        });
        onValue(ref(db, 'chamada_ativa'), snap => {
            const c = snap.val();
            if(c && c.status === "pendente") {
                document.getElementById('area-chamada').style.display = 'block';
                document.getElementById('esperando').style.display = 'none';
                document.getElementById('m-dest-p').innerText = c.destino.nome;
                document.getElementById('m-preco-p').innerText = c.valor;
                window.destGPS = c.destino;
            } else {
                document.getElementById('area-chamada').style.display = 'none';
                document.getElementById('esperando').style.display = 'block';
            }
        });
    };

    window.mudarMinhaFoto = () => {
        const url = prompt("Link da Foto (URL):");
        if(url && motoristaLogado) update(ref(db, 'motoristas/'+motoristaLogado), { foto: url });
    };

    window.aceitarCorrida = () => {
        update(ref(db, 'chamada_ativa'), { status: "aceita" });
        window.open(`https://www.google.com/maps/dir/?api=1&destination=${window.destGPS.lat},${window.destGPS.lon}&travelmode=motorcycle`);
    };

    // --- LÓGICA ADMINISTRADOR ---
    window.abrirAdmin = () => {
        const pass = prompt("Senha Mestra:");
        if(pass === "admin123") {
            mudarTela('painel-admin');
            onValue(ref(db, 'motoristas'), snap => {
                const list = document.getElementById('lista-motoristas');
                list.innerHTML = "";
                const data = snap.val();
                for(let id in data) {
                    let m = data[id];
                    list.innerHTML += `
                        <div class="admin-item">
                            <div style="text-align:left">
                                <strong>${m.nome}</strong><br>
                                <small>Saldo: R$ ${m.saldo.toFixed(2)}</small>
                            </div>
                            <div>
                                <button class="btn-sm" onclick="ajustarSaldo('${id}', 5)">+R$5</button>
                                <button class="btn-sm" onclick="ajustarSaldo('${id}', -5)">-R$5</button>
                            </div>
                        </div>`;
                }
            });
        } else { alert("Senha incorreta!"); }
    };

    window.ajustarSaldo = (id, valor) => {
        onValue(ref(db, `motoristas/${id}/saldo`), snap => {
            const atual = snap.val() || 0;
            update(ref(db, `motoristas/${id}`), { saldo: atual + valor });
        }, { onlyOnce: true });
    };
</script>
</body>
</html>
