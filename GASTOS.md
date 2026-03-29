layout: page
title: "Gastos Chat"
permalink: https://pharmagem-cto.github.io/GastosChat

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Gastos</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
      background: #f0f2f5;
      min-height: 100vh;
      display: flex;
      justify-content: center;
    }

    .app-container {
      width: 100%;
      max-width: 480px;
      background: #fff;
      display: flex;
      flex-direction: column;
      height: 100vh;
    }

    .header {
      background: #1a73e8;
      color: white;
      padding: 16px;
      text-align: center;
      flex-shrink: 0;
    }

    .header h1 {
      font-size: 20px;
      font-weight: 600;
      margin-bottom: 4px;
    }

    .header .period-info {
      font-size: 13px;
      opacity: 0.9;
    }

    .header .total-info {
      font-size: 14px;
      font-weight: 600;
      margin-top: 4px;
    }

    .chat-container {
      flex: 1;
      overflow-y: auto;
      padding: 16px;
      display: flex;
      flex-direction: column;
      gap: 12px;
    }

    .message {
      max-width: 80%;
      padding: 12px 16px;
      border-radius: 18px;
      font-size: 14px;
      line-height: 1.4;
      word-wrap: break-word;
    }

    .message.user {
      align-self: flex-end;
      background: #1a73e8;
      color: white;
      border-bottom-right-radius: 4px;
    }

    .message.app {
      align-self: flex-start;
      background: #e4e6eb;
      color: #050505;
      border-bottom-left-radius: 4px;
    }

    .message .summary-list {
      margin-top: 8px;
      padding-top: 8px;
      border-top: 1px solid rgba(0,0,0,0.1);
    }

    .message .summary-item {
      display: flex;
      justify-content: space-between;
      padding: 4px 0;
      font-size: 13px;
    }

    .message .summary-total {
      font-weight: 600;
      border-top: 1px solid rgba(0,0,0,0.15);
      margin-top: 6px;
      padding-top: 6px;
    }

    .input-container {
      padding: 12px 16px;
      background: #fff;
      border-top: 1px solid #e4e6eb;
      display: flex;
      gap: 8px;
      flex-shrink: 0;
    }

    .input-container input {
      flex: 1;
      padding: 12px 16px;
      border: 1px solid #e4e6eb;
      border-radius: 24px;
      font-size: 14px;
      outline: none;
    }

    .input-container input:focus {
      border-color: #1a73e8;
    }

    .input-container button {
      padding: 12px 20px;
      background: #1a73e8;
      color: white;
      border: none;
      border-radius: 24px;
      font-size: 14px;
      font-weight: 600;
      cursor: pointer;
    }

    .input-container button:hover {
      background: #1557b0;
    }

    .period-history-item {
      padding: 6px 0;
      border-bottom: 1px solid rgba(0,0,0,0.08);
      font-size: 13px;
    }

    .period-history-item:last-child {
      border-bottom: none;
    }
  </style>
</head>
<body>
  <div class="app-container">
    <div class="header">
      <h1>Gastos</h1>
      <div class="period-info" id="periodInfo">📅 Period: Loading...</div>
      <div class="total-info" id="totalInfo">Total: ₱0</div>
    </div>
    <div class="chat-container" id="chatContainer"></div>
    <div class="input-container">
      <input type="text" id="messageInput" placeholder="e.g., 500 rice and viand" autocomplete="off">
      <button id="sendBtn">Send</button>
    </div>
  </div>

  <script>
    // Storage key
    const STORAGE_KEY = 'gastos_expenses';

    // Get all expenses from localStorage
    function getExpenses() {
      const data = localStorage.getItem(STORAGE_KEY);
      return data ? JSON.parse(data) : [];
    }

    // Save expenses to localStorage
    function saveExpenses(expenses) {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(expenses));
    }

    // Generate UUID
    function generateUUID() {
      return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
        const r = Math.random() * 16 | 0;
        const v = c === 'x' ? r : (r & 0x3 | 0x8);
        return v.toString(16);
      });
    }

    // Get current period info based on date
    function getCurrentPeriodInfo(date = new Date()) {
      const day = date.getDate();
      const month = date.getMonth();
      const year = date.getFullYear();

      const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];

      if (day >= 10 && day <= 24) {
        // Period A: 10th to 24th
        const periodKey = `${year}-${String(month + 1).padStart(2, '0')}`;
        const periodLabel = `${monthNames[month]} 10–24`;
        return { periodKey, periodLabel, startDate: new Date(year, month, 10), endDate: new Date(year, month, 24) };
      } else {
        // Period B: 25th to 9th of next month
        let endMonth = month + 1;
        let endYear = year;
        if (endMonth > 11) {
          endMonth = 0;
          endYear++;
        }
        const periodKey = `${year}-${String(month + 1).padStart(2, '0')}`;
        const periodLabel = `${monthNames[month]} 25–${monthNames[endMonth]} 9`;
        const startDate = new Date(year, month, 25);
        const endDate = new Date(endYear, endMonth, 9);
        return { periodKey, periodLabel, startDate, endDate };
      }
    }

    // Check if a date falls within a period
    function isInPeriod(date, periodInfo) {
      const ts = date.getTime();
      return ts >= periodInfo.startDate.getTime() && ts <= periodInfo.endDate.getTime();
    }

    // Get expenses for current period
    function getCurrentPeriodExpenses() {
      const expenses = getExpenses();
      const currentPeriod = getCurrentPeriodInfo();
      return expenses.filter(exp => exp.period === currentPeriod.periodKey);
    }

    // Calculate total for current period
    function getCurrentPeriodTotal() {
      const expenses = getCurrentPeriodExpenses();
      return expenses.reduce((sum, exp) => sum + exp.amount, 0);
    }

    // Parse user message to extract amount and description
    function parseMessage(message) {
      const trimmed = message.trim();
      if (!trimmed) return null;

      // Try to find a number at the beginning or anywhere in the message
      const match = trimmed.match(/^(\d+(?:,\d{3})*(?:\.\d{2})?)\s+(.+)$/);
      if (match) {
        const amount = parseFloat(match[1].replace(/,/g, ''));
        const description = match[2].trim();
        return { amount, description };
      }

      // Try to find any number in the message
      const anyMatch = trimmed.match(/(\d+(?:,\d{3})*(?:\.\d{2})?)/);
      if (anyMatch) {
        const amount = parseFloat(anyMatch[1].replace(/,/g, ''));
        const description = trimmed.replace(anyMatch[0], '').trim().replace(/\s+/g, ' ');
        return { amount, description };
      }

      return null;
    }

    // Format currency
    function formatCurrency(amount) {
      return '₱' + amount.toLocaleString('en-PH', { minimumFractionDigits: 0, maximumFractionDigits: 2 });
    }

    // Add message to chat
    function addMessage(text, isUser = false) {
      const chatContainer = document.getElementById('chatContainer');
      const messageDiv = document.createElement('div');
      messageDiv.className = `message ${isUser ? 'user' : 'app'}`;
      messageDiv.textContent = text;
      chatContainer.appendChild(messageDiv);
      chatContainer.scrollTop = chatContainer.scrollHeight;
    }

    // Add HTML message to chat
    function addHTMLMessage(html, isUser = false) {
      const chatContainer = document.getElementById('chatContainer');
      const messageDiv = document.createElement('div');
      messageDiv.className = `message ${isUser ? 'user' : 'app'}`;
      messageDiv.innerHTML = html;
      chatContainer.appendChild(messageDiv);
      chatContainer.scrollTop = chatContainer.scrollHeight;
    }

    // Update header info
    function updateHeader() {
      const periodInfo = getCurrentPeriodInfo();
      const total = getCurrentPeriodTotal();

      document.getElementById('periodInfo').textContent = `📅 Period: ${periodInfo.periodLabel}`;
      document.getElementById('totalInfo').textContent = `Total: ${formatCurrency(total)}`;
    }

    // Handle expense entry
    function handleExpense(message) {
      const parsed = parseMessage(message);
      
      if (!parsed) {
        addMessage("I couldn't read an amount there. Try: [amount] [description]");
        return;
      }

      const currentPeriod = getCurrentPeriodInfo();
      const expense = {
        id: generateUUID(),
        amount: parsed.amount,
        description: parsed.description,
        period: currentPeriod.periodKey,
        periodLabel: currentPeriod.periodLabel,
        timestamp: new Date().toISOString()
      };

      const expenses = getExpenses();
      expenses.push(expense);
      saveExpenses(expenses);

      const newTotal = getCurrentPeriodTotal();
      addMessage(`Got it! ${formatCurrency(parsed.amount)} for ${parsed.description}. Period total: ${formatCurrency(newTotal)}.`);
      updateHeader();
    }

    // Handle summary command
    function handleSummary() {
      const expenses = getCurrentPeriodExpenses();
      const currentPeriod = getCurrentPeriodInfo();

      if (expenses.length === 0) {
        addMessage(`No expenses recorded for ${currentPeriod.periodLabel}.`);
        return;
      }

      const total = expenses.reduce((sum, exp) => sum + exp.amount, 0);
      
      let html = `<strong>${currentPeriod.periodLabel}</strong><div class="summary-list">`;
      expenses.forEach(exp => {
        html += `<div class="summary-item"><span>${exp.description}</span><span>${formatCurrency(exp.amount)}</span></div>`;
      });
      html += `<div class="summary-item summary-total"><span>Total</span><span>${formatCurrency(total)}</span></div></div>`;
      
      addHTMLMessage(html);
    }

    // Handle history command
    function handleHistory() {
      const expenses = getExpenses();
      
      if (expenses.length === 0) {
        addMessage("No expense history found.");
        return;
      }

      // Group by period
      const periodMap = {};
      expenses.forEach(exp => {
        if (!periodMap[exp.period]) {
          periodMap[exp.period] = { label: exp.periodLabel, total: 0, count: 0 };
        }
        periodMap[exp.period].total += exp.amount;
        periodMap[exp.period].count++;
      });

      // Sort periods (newest first)
      const sortedPeriods = Object.keys(periodMap).sort().reverse();

      let html = `<strong>All Periods</strong><div class="summary-list">`;
      sortedPeriods.forEach(periodKey => {
        const info = periodMap[periodKey];
        html += `<div class="period-history-item"><span>${info.label} (${info.count} items)</span><span>${formatCurrency(info.total)}</span></div>`;
      });
      html += `</div>`;

      addHTMLMessage(html);
    }

    // Handle user input
    function handleInput() {
      const input = document.getElementById('messageInput');
      const message = input.value.trim();
      
      if (!message) return;

      addMessage(message, true);
      input.value = '';

      const lowerMessage = message.toLowerCase();
      
      if (lowerMessage === 'summary') {
        handleSummary();
      } else if (lowerMessage === 'history') {
        handleHistory();
      } else {
        handleExpense(message);
      }
    }

    // Load chat history for current period
    function loadChatHistory() {
      const expenses = getCurrentPeriodExpenses();
      
      if (expenses.length === 0) {
        addMessage("Welcome to Gastos! Start by typing an expense like '500 rice and viand'. Type 'summary' to see your expenses or 'history' for all periods.");
        return;
      }

      expenses.forEach(exp => {
        const time = new Date(exp.timestamp).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
        addMessage(`${time}: ${formatCurrency(exp.amount)} - ${exp.description}`, true);
      });

      addMessage(`Loaded ${expenses.length} expense(s) for this period.`);
    }

    // Initialize app
    function init() {
      updateHeader();
      loadChatHistory();

      const input = document.getElementById('messageInput');
      const sendBtn = document.getElementById('sendBtn');

      input.addEventListener('keypress', (e) => {
        if (e.key === 'Enter') {
          handleInput();
        }
      });

      sendBtn.addEventListener('click', handleInput);
    }

    // Start the app
    init();
  </script>
</body>
</html>
