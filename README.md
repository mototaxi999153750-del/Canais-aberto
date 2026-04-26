<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Moto Táxi Aquidabã</title>
    <style>
        :root { --green: #22c55e; --blue: #3b82f6; --dark: #0f172a; --card: #1e293b; }
        body { background: var(--dark); color: white; font-family: sans-serif; margin: 0; padding: 0; overflow-x: hidden; }
        
        /* ESTILO DO MENU RAIZ */
        #menu-raiz { display: flex; flex-direction: column; align-items: center; justify-content: center; min-height: 100vh; padding: 20px; }
        .btn-link { width: 100%; max-width: 350px; background: var(--card); color: white; text-decoration: none; padding: 20px; margin-bottom: 15px; border-radius: 18px; border: 1px solid #334155; font-weight: bold; text-align: center; font-size: 18px; }
        .btn-link.destaque { border-color: var(--green); background: rgba(34, 197, 94, 0.05); }

        /* ESTILO DO SISTEMA DE CÁLCULO */
        #sistema-calculo { display: none; padding: 20px; min-height: 100vh; flex-direction: column; align-items: center; justify-content: center; }
        .card { background: var(--card); padding: 25px; border-radius: 25px; width: 100%; max-width: 380px; box-sizing: border-box; border: 1px solid #334155; text-align: center; }
        
        input { width: 100%; padding: 16px; margin: 12px 0; border-radius: 12px; border: 1px solid #334155; background: var(--dark); color: white; box-sizing: border-box; font-size: 16px; }
        button { width: 100%; padding: 18px; border-radius: 12px; border: none; font-weight: bold; cursor: pointer; margin-top: 10px; font-size: 16px; transition: 0.2s; }
        .btn-gps { background: var(--blue); color: white; }
        .btn-calc { background: var(--green); color: white; }
        
        #box-res { display: none; margin-top: 25px; padding: 15px; border: 2px dashed var(--green); border-radius: 15px; background: rgba(34, 197, 94, 0.1); }
        .preco { font-size: 36px; font-weight: 800; color: var(--green); margin: 5px 0; display: block; }

        /* Estilo Google Autocomplete */
        .pac-container { background: var(--card) !important; border: 1px solid #334155 !important; border-radius: 10px; z-index: 9999 !important; color: white !important; }
        .pac-item { color: #94a3b8 !important; border-top: 1px solid #334155 !important; padding: 10px; }
        .pac-item-query { color: white !important; }
    </style>
</head>
<body>

    <div id="menu-raiz">
        <div style="font-size: 60px;">🏍️</div>
        <h2 style="color: var(--green); margin-bottom: 40px;">Moto Táxi Aquidabã</h2>
        
        <a href="javascript:void(0)" class="btn-link destaque" onclick="abrirCalculadora()">💰 APP DO CLIENTE</a>
        <a href="https://wa.me/5579999999999" class="btn-link">📲 SUPORTE / GERENTE</a>
        <a href="#" class="btn-link">⚙️ PAINEL DO PILOTO</a>
    </div>

    <div id="sistema-calculo">
        <div class="card">
            <h2 style="color: var(--green); margin-bottom: 5px;">Calcular Corrida</h2>
            <p style="color: #94a3b8; font-size: 13px; margin-bottom: 20px;">Valor real por distância percorrida</p>

            <button class="btn-gps" onclick="pegarLocalizacao()">📍 1. Confirmar Minha Origem</button>
            <div id="st-gps" style="font-size:12px; color:#fbbf24; margin-top:8px;">Aguardando sinal GPS...</div>

            <input type="text" id="destino" placeholder="2. Digite o endereço de destino">

            <button class="btn-calc" onclick="calcularValor()">💰 3. Calcular Valor</button>

            <div id="box-res">
                <span id="txt-km" style="color:#94a3b8; font-size:14px;"></span>
                <span class="preco" id="txt-preco">R$ 0,00</span>
                <button style="background:#25d366; color:white;" onclick="enviarWhatsApp()">📲 Solicitar via WhatsApp</button>
            </div>
            
            <button style="background:transparent; color:#94a3b8; font-size:12px; text-decoration:underline;" onclick="voltarMenu()">← Voltar ao Início</button>
        </div>
    </div>

    <script src="https://maps.googleapis.com/maps/api/js?key=AIzaSyCYjEDrKNcWwPBKrBbOOvORNqsfu_SB9EQ&libraries=places"></script>

    <script>
        let lat, lng, kmFinal, precoFinal;
        const inputDest = document.getElementById('destino');
        const auto = new google.maps.places.Autocomplete(inputDest);

        function abrirCalculadora() {
            document.getElementById('menu-raiz').style.display = 'none';
            document.getElementById('sistema-calculo').style.display = 'flex';
        }

        function voltarMenu() {
            document.getElementById('menu-raiz').style.display = 'flex';
            document.getElementById('sistema-calculo').style.display = 'none';
        }

        function pegarLocalizacao() {
            navigator.geolocation.getCurrentPosition(p => {
                lat = p.coords.latitude; lng = p.coords.longitude;
                document.getElementById('st-gps').innerText = "Localização confirmada ✔️";
                document.getElementById('st-gps').style.color = "#22c55e";
            }, () => alert("Ative seu GPS!"));
        }

        function calcularValor() {
            const dest = inputDest.value;
            if (!lat || !dest) return alert("Ative o GPS e digite o destino!");

            const service = new google.maps.DistanceMatrixService();
            service.getDistanceMatrix({
                origins: [{lat, lng}], destinations: [dest], travelMode: 'DRIVING'
            }, (r, s) => {
                if (s === 'OK') {
                    const el = r.rows[0].elements[0];
                    if (el.status === 'OK') {
                        kmFinal = el.distance.value / 1000;
                        // REGRA: R$ 5,00 até 2km + 1,50/km adicional
                        precoFinal = (kmFinal <= 2) ? 5.00 : 5.00 + ((kmFinal - 2) * 1.50);
                        
                        document.getElementById('txt-km').innerText = "Distância: " + kmFinal.toFixed(2) + " km";
                        document.getElementById('txt-preco').innerText = "R$ " + precoFinal.toFixed(2);
                        document.getElementById('box-res').style.display = 'block';
                    } else { alert("Local não encontrado."); }
                }
            });
        }

        function enviarWhatsApp() {
            const dest = inputDest.value;
            const tel = "5579999999999";
            const msg = `*NOVO PEDIDO - AQUIDABÃ*%0A📍 Local: https://www.google.com/maps?q=${lat},${lng}%0A🏁 Destino: ${dest}%0A📏 Km: ${kmFinal.toFixed(2)}%0A💰 Valor: R$ ${precoFinal.toFixed(2)}`;
            window.open(`https://wa.me/${tel}?text=${msg}`);
        }
    </script>
</body>
</html>
