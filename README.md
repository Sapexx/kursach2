# kursach2
# test_site.py
from flask import Flask, request, jsonify
import socket
import os

app = Flask(__name__)

@app.route("/")
def index():
    # --- 1. Что видит сервер о клиенте ---
    remote_addr = request.remote_addr  # IP клиента (или прокси)
    xff = request.headers.get("X-Forwarded-For", "")  # реальный IP, если прокси его передаёт
    real_ip = xff.split(",")[0].strip() if xff else remote_addr

    # --- 2. Обратный DNS (может не работать, если PTR не настроен) ---
    try:
        reverse_dns = socket.gethostbyaddr(real_ip)[0]
    except Exception as e:
        reverse_dns = f"ОШИБКА: {e}"

    # --- 3. Переменные окружения WSGI (что передал IIS/прокси/веб-сервер) ---
    wsgi_vars = {}
    for key, value in request.environ.items():
        if isinstance(value, (str, int, float)) or value is None:
            wsgi_vars[key] = str(value)

    # --- 4. Все HTTP-заголовки запроса ---
    headers = dict(request.headers)

    # --- 5. Что удалось получить из аутентификации ---
    auth_info = {
        "REMOTE_USER": request.environ.get("REMOTE_USER"),
        "AUTH_USER": request.environ.get("AUTH_USER"),
        "LOGON_USER": request.environ.get("LOGON_USER"),
        "REMOTE_HOST": request.environ.get("REMOTE_HOST"),
        "REMOTE_ADDR": request.environ.get("REMOTE_ADDR"),
        "HTTP_AUTHORIZATION": request.headers.get("Authorization", "")[:50] + "...",
    }

    # --- 6. Собираем всё в один ответ ---
    result = {
        "1_IP_информация": {
            "remote_addr": remote_addr,
            "x_forwarded_for": xff,
            "реальный_ip_клиента": real_ip,
        },
        "2_обратный_DNS": {
            "результат": reverse_dns,
        },
        "3_аутентификация": auth_info,
        "4_заголовки_запроса": headers,
        "5_переменные_WSGI": wsgi_vars,
    }

    return jsonify(result)

# Отдельная страница для удобного просмотра в браузере
@app.route("/view")
def view():
    return """
    <html><head><meta charset="utf-8"><title>Диагностика</title></head>
    <body>
        <h1>Диагностика клиента</h1>
        <p>Откройте <a href="/">/</a> для JSON или нажмите кнопку:</p>
        <button onclick="load()">Загрузить данные</button>
        <pre id="out" style="background:#f4f4f4;padding:10px;"></pre>
        <script>
        async function load() {
            const r = await fetch('/');
            const data = await r.json();
            document.getElementById('out').textContent = JSON.stringify(data, null, 2);
        }
        load();
        </script>
    </body></html>
    """

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080, debug=True)
