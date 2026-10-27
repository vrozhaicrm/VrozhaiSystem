<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Овощная лавка</title>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        .tab { display: none; }
        .active { display: block; }
        button { padding: 8px; margin: 5px; cursor: pointer; }
        table { width: 100%; border-collapse: collapse; margin-top: 10px; }
        th, td { border: 1px solid #ccc; padding: 10px; text-align: left; }
        nav { margin-bottom: 20px; }
        nav button { background: #f0f0f0; border: 1px solid #ccc; font-size: 16px; }
    </style>
</head>
<body>

    <nav>
        <button onclick="showTab('stock')">Склад</button>
        <button onclick="showTab('supply')">Завоз</button>
        <button onclick="showTab('sales')">Продажи</button>
        <button onclick="showTab('accounting')">Бухгалтерия</button>
    </nav>

    <!-- СКЛАД -->
    <div id="stock" class="tab active">
        <h2>Склад</h2>
        <input type="text" id="search-stock" placeholder="Поиск товара..." oninput="updateUI()">
        <table>
            <thead><tr><th>Товар</th><th>Кол-во (кг)</th><th>Действия</th></tr></thead>
            <tbody id="stock-list"></tbody>
        </table>
    </div>

    <!-- ЗАВОЗ -->
    <div id="supply" class="tab">
        <h2>Завоз товара</h2>
        <input type="text" id="sup-item" placeholder="Название товара">
        <input type="number" id="sup-qty" placeholder="Кг">
        <input type="number" id="sup-price" placeholder="Цена закупки (грн/кг)">
        <button onclick="addSupply()">Оприходовать</button>
        <h3>История завозов:</h3>
        <input type="text" id="search-supply" placeholder="Поиск по завозам..." oninput="updateUI()">
        <table>
            <thead><tr><th>Дата</th><th>Товар</th><th>Кол-во</th><th>Цена</th><th>Стоимость</th><th>Действия</th></tr></thead>
            <tbody id="supply-list"></tbody>
        </table>
    </div>

    <!-- ПРОДАЖИ -->
    <div id="sales" class="tab">
        <h2>Продажи</h2>
        <label>Выберите товар:</label>
        <select id="sell-item"></select>
        <input type="number" id="sell-qty" placeholder="Кг">
        <input type="number" id="sell-price" placeholder="Сумма (грн)">
        <button onclick="sellItem()">Продать</button>

        <h3>История продаж:</h3>
        <input type="text" id="search-sales" placeholder="Поиск по продажам..." oninput="updateUI()">
        <table>
            <thead><tr><th>Товар</th><th>Вес</th><th>Цена за 1 кг</th><th>Сумма</th><th>Время</th><th>Действия</th></tr></thead>
            <tbody id="recent-sales-list"></tbody>
        </table>
<button id="load-more-btn-sales" onclick="loadMoreSales()" style="display:none;">Загрузить еще</button>
</div>

    <div id="accounting" class="tab">
    <h2>Бухгалтерия</h2>
    <div style="background: #f9f9f9; padding: 15px; border-radius: 5px;">
        <label>Режим:</label>
        <select id="filter-mode" onchange="updateFilterInput()">
            <option value="day">День</option>
            <option value="month">Месяц</option>
            <option value="year">Год</option>
        </select>
        
        <!-- Контейнер, куда мы будем вставлять нужный тип инпута -->
        <span id="filter-input-container">
            <input type="date" id="filter-value" onchange="filterAccounting()">
        </span>
    </div>

    <h3>Итого за период:</h3>
    <p>Доход: <b><span id="total-income">0</span></b> грн | 
       Расход: <b><span id="total-expense">0</span></b> грн | 
       Прибыль: <b><span id="total-profit">0</span></b> грн</p>
    
    <table>
        <thead><tr><th>Товар</th><th>Вес</th><th>Цена/кг</th><th>Сумма продажи</th><th>Дата</th></tr></thead>
        <tbody id="accounting-list"></tbody>
    </table>
</div>

    <script>
    const GOOGLE_URL = "https://script.google.com/macros/s/AKfycbw-PgmkI5M3MvzBK2Xl9JcD7vNBTS-eCUV1-9ykpBy5Wi4doDshV7vc_WCgKf51E_ZGSA/exec";

        function sendToGoogle(payload) {
            fetch(GOOGLE_URL, {
                method: "POST",
                mode: "no-cors",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify(payload)
            });
        }
        let inventory = JSON.parse(localStorage.getItem('inventory')) || {};
        let history = JSON.parse(localStorage.getItem('history')) || [];
        let supplies = JSON.parse(localStorage.getItem('supplies')) || [];
        let salesLimit = 30;

function loadMoreSales() {
    salesLimit += 30;
    updateUI();
}
        function showTab(tabId) {
            document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
            document.getElementById(tabId).classList.add('active');
        }

        function updateUI() {
    // 1. Склад
    let list = document.getElementById('stock-list');
    list.innerHTML = "";
    let select = document.getElementById('sell-item');
    select.innerHTML = '<option value="">-- Выберите --</option>';

let searchStock = document.getElementById('search-stock').value.toLowerCase();
    for (let item in inventory) {
if (!item.includes(searchStock)) continue;
        select.innerHTML += `<option value="${item}">${item.toUpperCase()}</option>`;
        let data = inventory[item];
        let colorStyle = data.kg < 5 ? "color: red; font-weight: bold;" : "";
        list.innerHTML += `<tr>
            <td>${item.toUpperCase()}</td>
            <td style="${colorStyle}">${data.kg} кг</td>
            <td>
                <button onclick="editItem('${item}')">Ред.</button>
                <button onclick="deleteItem('${item}')" style="color:red">Удалить</button>
            </td>
        </tr>`;
    }

    // 2. Завоз
        let searchSupply = document.getElementById('search-supply').value.toLowerCase();
    document.getElementById('supply-list').innerHTML = supplies.slice().reverse()
        .filter(s => s.item.toLowerCase().includes(searchSupply))
        .map((s, index) => {
            // Реальный индекс в исходном массиве supplies
            let realIndex = supplies.length - 1 - index;
            return `<tr>
                <td>${s.date}</td>
                <td>${s.item.toUpperCase()}</td>
                <td>${s.qty} кг</td>
                <td>${s.price} грн</td>
                <td>${(s.qty * s.price).toFixed(2)} грн</td>
                <td>
                    <button onclick="editSupply(${realIndex})">Ред.</button>
                    <button onclick="deleteSupply(${realIndex})" style="color:red">Удалить</button>
                </td>
            </tr>`;
        })
        .join('');

    // 3. Продажи
    let searchSales = document.getElementById('search-sales').value.toLowerCase();
    let filteredSales = history.slice().reverse().filter(h => h.item.toLowerCase().includes(searchSales));
    
    document.getElementById('recent-sales-list').innerHTML = filteredSales
        .slice(0, salesLimit) 
        .map((h, index) => {
            let realIndex = history.length - 1 - index;
            return `<tr>
                <td>${h.item.toUpperCase()}</td>
                <td>${h.qty} кг</td>
                <td>${(h.pricePerKg || 0).toFixed(2)} грн</td>
                <td>${(h.totalSum || 0).toFixed(2)} грн</td>
                <td>${h.date}</td>
                <td>
                    <button onclick="editSale(${realIndex})">Ред.</button>
                    <button onclick="deleteSale(${realIndex})" style="color:red">Удалить</button>
                </td>
            </tr>`;
        })
        .join('');

    // Показываем кнопку, если записей еще много
    document.getElementById('load-more-btn-sales').style.display = (filteredSales.length > salesLimit) ? "inline-block" : "none";

    localStorage.setItem('inventory', JSON.stringify(inventory));
    localStorage.setItem('history', JSON.stringify(history));
    localStorage.setItem('supplies', JSON.stringify(supplies));
}

        function addSupply() {
            let item = document.getElementById('sup-item').value.trim().toLowerCase();
            let qty = parseFloat(document.getElementById('sup-qty').value);
            let price = parseFloat(document.getElementById('sup-price').value);
            if (!item || isNaN(qty) || isNaN(price)) return alert("Заполните все поля!");

            if (!inventory[item]) inventory[item] = { kg: 0, buyPrice: price };
            inventory[item].kg += qty;
            inventory[item].buyPrice = price;

            let date = new Date().toLocaleString();
            supplies.push({ date: date, item, qty, price });
            sendToGoogle({ type: "supply", date: date, item: item, qty: qty, price: price, total: (qty * price).toFixed(2) });
            sendToGoogle({ type: "sync_stock", inventory: inventory });
            document.getElementById('sup-item').value = "";
            document.getElementById('sup-qty').value = "";
            document.getElementById('sup-price').value = "";
            updateUI();
        }

        function sellItem() {
    let item = document.getElementById('sell-item').value;
    let qty = parseFloat(document.getElementById('sell-qty').value);
    let pricePerKg = parseFloat(document.getElementById('sell-price').value);

    if (!item || isNaN(qty) || isNaN(pricePerKg) || inventory[item].kg < qty) {
        return alert("Ошибка данных или нет товара!");
    }

    let totalSum = qty * pricePerKg; // Считаем общую сумму
    inventory[item].kg -= qty;

    let date = new Date().toLocaleString();
    // ВАЖНО: используем именно эти имена полей
    history.push({ 
        item: item, 
        qty: qty, 
        pricePerKg: pricePerKg, 
        totalSum: totalSum, 
        date: new Date().toLocaleString() 
    });

    sendToGoogle({ type: "sale", date: date, item: item, qty: qty, pricePerKg: pricePerKg, totalSum: totalSum.toFixed(2) });
    sendToGoogle({ type: "sync_stock", inventory: inventory });

    document.getElementById('sell-qty').value = "";
    document.getElementById('sell-price').value = "";
    updateUI();
}

        function deleteItem(item) {
            if (confirm(`Удалить ${item}?`)) { delete inventory[item]; updateUI(); }
        }

        // УДАЛЕНИЕ И РЕДАКТИРОВАНИЕ ЗАВОЗА
        function deleteSupply(index) {
            let s = supplies[index];
            if (confirm(`Удалить завоз товара ${s.item.toUpperCase()} на ${s.qty} кг?`)) {
                // Возвращаем остаток склада (списываем ошибочный завоз)
                if (inventory[s.item]) {
                    inventory[s.item].kg -= s.qty;
                    if (inventory[s.item].kg < 0) inventory[s.item].kg = 0;
                }
                supplies.splice(index, 1);
                sendToGoogle({ type: "sync_stock", inventory: inventory });
                updateUI();
            }
        }

        function editSupply(index) {
            let s = supplies[index];
            let newQty = prompt(`Введите новый вес (кг) для ${s.item.toUpperCase()}:`, s.qty);
            let newPrice = prompt(`Введите новую цену закупки (грн/кг):`, s.price);

            if (newQty !== null && newPrice !== null && !isNaN(newQty) && !isNaN(newPrice)) {
                let diffKg = parseFloat(newQty) - s.qty; // разница в весе
                s.qty = parseFloat(newQty);
                s.price = parseFloat(newPrice);
                
                // Корректируем склад на разницу
                if (inventory[s.item]) {
                    inventory[s.item].kg += diffKg;
                    inventory[s.item].buyPrice = s.price;
                }
                
                sendToGoogle({ type: "sync_stock", inventory: inventory });
                updateUI();
            }
        }

        // УДАЛЕНИЕ И РЕДАКТИРОВАНИЕ ПРОДАЖИ
        function deleteSale(index) {
            let h = history[index];
            if (confirm(`Удалить продажу товара ${h.item.toUpperCase()} на ${h.qty} кг?`)) {
                // Возвращаем проданный товар обратно на склад
                if (inventory[h.item]) {
                    inventory[h.item].kg += h.qty;
                }
                history.splice(index, 1);
                sendToGoogle({ type: "sync_stock", inventory: inventory });
                updateUI();
            }
        }

        function editSale(index) {
            let h = history[index];
            let newQty = prompt(`Введите новый вес продажи (кг) для ${h.item.toUpperCase()}:`, h.qty);
            let newPrice = prompt(`Введите новую цену продажи за 1 кг (грн):`, h.pricePerKg);

            if (newQty !== null && newPrice !== null && !isNaN(newQty) && !isNaN(newPrice)) {
                let parsedQty = parseFloat(newQty);
                let parsedPrice = parseFloat(newPrice);
                let diffKg = parsedQty - h.qty;

                // Проверяем, хватит ли товара на складе, если увеличиваем продажу
                if (diffKg > 0 && inventory[h.item].kg < diffKg) {
                    return alert("На складе недостаточно товара для увеличения продажи!");
                }

                // Корректируем склад
                if (inventory[h.item]) {
                    inventory[h.item].kg -= diffKg;
                }

                h.qty = parsedQty;
                h.pricePerKg = parsedPrice;
                h.totalSum = parsedQty * parsedPrice;

                sendToGoogle({ type: "sync_stock", inventory: inventory });
                updateUI();
            }
        }

function editItem(item) {
    // Получаем текущее значение кг (безопасно, даже если данные в другом формате)
    let currentKg = (typeof inventory[item] === 'object') ? inventory[item].kg : inventory[item];
    
    let newQty = prompt(`Введите новое количество для ${item.toUpperCase()}:`, currentKg);
    
    if (newQty !== null && !isNaN(newQty)) {
        let value = parseFloat(newQty);
        
        // Если это объект, обновляем поле kg, если число — перезаписываем
        if (typeof inventory[item] === 'object') {
            inventory[item].kg = value;
        } else {
            inventory[item] = { kg: value, buyPrice: 0 };
        }
        
        updateUI();
    }
}

        function filterAccounting() {
    let mode = document.getElementById('filter-mode').value;
    let inputEl = document.getElementById('filter-value');
    if (!inputEl) return;
    let val = inputEl.value; 
    let list = document.getElementById('accounting-list');
    let income = 0, expense = 0;
    list.innerHTML = "";
    
    if (!val) return;

    history.forEach(h => {
        let d = h.date.split(',')[0].trim().split('.'); 
        let yyyy_mm_dd = `${d[2]}-${d[1].padStart(2, '0')}-${d[0].padStart(2, '0')}`;
        let yyyy_mm = `${d[2]}-${d[1].padStart(2, '0')}`;
        let yyyy = d[2];

        let match = false;
        if (mode === 'day' && yyyy_mm_dd === val) match = true;
        else if (mode === 'month' && yyyy_mm === val) match = true;
        else if (mode === 'year' && yyyy === val) match = true;

        if (match) {
            let buyPrice = (inventory[h.item] && inventory[h.item].buyPrice) ? inventory[h.item].buyPrice : 0;
            
            // Используем h.totalSum и h.qty
            let currentSaleSum = parseFloat(h.totalSum) || 0;
            let currentQty = parseFloat(h.qty) || 0;
            
            income += currentSaleSum;
            expense += (currentQty * buyPrice);
            
            // Вывод в таблицу (исправлены переменные)
            list.innerHTML += `<tr>
                <td>${h.item.toUpperCase()}</td>
                <td>${currentQty} кг</td>
                <td>${(parseFloat(h.pricePerKg) || 0).toFixed(2)} грн</td>
                <td>${currentSaleSum.toFixed(2)} грн</td>
                <td>${h.date}</td>
            </tr>`;
        }
    });

    document.getElementById('total-income').innerText = income.toFixed(2);
    document.getElementById('total-expense').innerText = expense.toFixed(2);
    document.getElementById('total-profit').innerText = (income - expense).toFixed(2);
}

        updateUI();

function updateFilterInput() {
    let mode = document.getElementById('filter-mode').value;
    let container = document.getElementById('filter-input-container');
    
    if (mode === 'day') {
        container.innerHTML = `<input type="date" id="filter-value" onchange="filterAccounting()">`;
    } else if (mode === 'month') {
        container.innerHTML = `<input type="month" id="filter-value" onchange="filterAccounting()">`;
    } else if (mode === 'year') {
        container.innerHTML = `<input type="number" id="filter-value" value="2024" min="2000" max="2099" oninput="filterAccounting()">`;
    }
    // Сброс итогов при смене режима
    document.getElementById('accounting-list').innerHTML = "";
    document.getElementById('total-income').innerText = "0";
    document.getElementById('total-expense').innerText = "0";
    document.getElementById('total-profit').innerText = "0";
}
</script>

    <!-- Кнопка проверки связи с Google -->
<div style="text-align: center; margin: 30px;">
  <button id="testBtn" style="padding: 12px 20px; font-size: 16px; cursor: pointer; background: #28a745; color: white; border: none; border-radius: 5px;">
    Проверить связь с Google Таблицей
  </button>
</div>

<script>
  document.getElementById('testBtn').addEventListener('click', function() {
    const googleUrl = "https://script.google.com/macros/s/AKfycbw-PgmkI5M3MvzBK2Xl9JcD7vNBTS-eCUV1-9ykpBy5Wi4doDshV7vc_WCgKf51E_ZGSA/exec"; // <-- Вставь сюда ссылку /exec
    
    this.innerText = "Отправка...";
    
    fetch(googleUrl, {
      method: "POST",
      mode: "no-cors",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text: "Тестовая запись из CRM на GitHub!" })
    })
    .then(() => {
      alert("Данные отправлены! Проверь свою Google Таблицу.");
      this.innerText = "Проверить связь с Google Таблицей";
    })
    .catch(err => {
      alert("Ошибка отправки: " + err);
      this.innerText = "Ошибка";
    });
  });
</script>
    
</body>
</html>
