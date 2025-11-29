<html lang="th">
 <head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ระบบติดตามเอกสารโครงการกิจกรรม</title>
  <script src="/_sdk/data_sdk.js"></script>
  <script src="/_sdk/element_sdk.js"></script>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    body {
      box-sizing: border-box;
    }
    
    * {
      font-family: 'Sarabun', 'Noto Sans Thai', sans-serif;
    }
    
    .status-badge {
      padding: 0.25rem 0.75rem;
      border-radius: 9999px;
      font-size: 0.875rem;
      font-weight: 500;
    }
    
    .notification-dot {
      width: 8px;
      height: 8px;
      background-color: #ef4444;
      border-radius: 50%;
      position: absolute;
      top: 0;
      right: 0;
    }
  </style>
  <style>@view-transition { navigation: auto; }</style>
 </head>
 <body class="w-full min-h-full">
  <div id="app" class="w-full min-h-full"></div>
  <script>
    const defaultConfig = {
      system_title: "ระบบติดตามเอกสารโครงการกิจกรรม",
      organization_name: "หน่วยงาน",
      primary_color: "#1e40af",
      secondary_color: "#f3f4f6",
      text_color: "#1f2937",
      success_color: "#10b981",
      warning_color: "#f59e0b"
    };

    let currentUser = null;
    let allRecords = [];
    let showingNotifications = false;

    let systemUsers = {
      staff: [
        { id: "staff-1", username: "admin", password: "admin123", name: "ผู้ดูแลระบบ", role: "staff", position: "แอดมิน" },
        { id: "staff-2", username: "finance", password: "finance123", name: "เจ้าหน้าที่การเงิน", role: "staff", position: "การเงิน" },
        { id: "staff-3", username: "planning", password: "plan123", name: "เจ้าหน้าที่แผนงาน", role: "staff", position: "แผนงาน" },
        { id: "staff-4", username: "supply", password: "supply123", name: "เจ้าหน้าที่พัสดุ", role: "staff", position: "พัสดุ" }
      ],
      departments: [
        { id: "dept-1", username: "curriculum", password: "dept123", name: "งานหลักสูตร", department: "งานหลักสูตร", role: "department" },
        { id: "dept-2", username: "manpower", password: "dept123", name: "งานอัตรากำลัง", department: "งานอัตรากำลัง", role: "department" },
        { id: "dept-3", username: "supervision", password: "dept123", name: "งานนิเทศการเรียนการสอน", department: "งานนิเทศการเรียนการสอน", role: "department" },
        { id: "dept-4", username: "building", password: "dept123", name: "งานอาคารสถานที่", department: "งานอาคารสถานที่", role: "department" },
        { id: "dept-5", username: "document", password: "dept123", name: "งานสารบรรณ", department: "งานสารบรรณ", role: "department" },
        { id: "dept-6", username: "discipline", password: "dept123", name: "งานวินัยนักเรียน", department: "งานวินัยนักเรียน", role: "department" }
      ]
    };

    const STATUSES = [
      "เสนอ",
      "ดำเนินกิจกรรม",
      "รอแก้ไข",
      "ขอเบิก",
      "รอแก้ไข-ตั้งเบิก",
      "เสนอผู้บริหาร",
      "อนุมัติเบิก",
      "จ่ายเงินเสร็จสิ้น"
    ];

    function generateDocumentNumber() {
      const requests = allRecords.filter(r => r.type === "request");
      if (requests.length === 0) return "1";
      
      const numbers = requests
        .map(r => parseInt(r.document_number))
        .filter(n => !isNaN(n));
      
      if (numbers.length === 0) return "1";
      
      const maxNumber = Math.max(...numbers);
      return String(maxNumber + 1);
    }

    function getStatusColor(status) {
      const colors = {
        "เ��นอ": "#3b82f6",
        "ดำเนินกิจกรรม": "#f59e0b",
        "รอแก้ไข": "#ef4444",
        "ขอเบิก": "#ec4899",
        "รอแก้ไข-ตั้งเบิก": "#dc2626",
        "เสนอผู้บริหาร": "#14b8a6",
        "อนุมัติเบิก": "#10b981",
        "จ่ายเงินเสร็จสิ้น": "#22c55e"
      };
      return colors[status] || "#6b7280";
    }

    let viewedNotifications = new Set();

    function getNotifications() {
      if (!currentUser) return [];
      
      if (currentUser.role === "staff") {
        return allRecords.filter(r => {
          if (r.type !== "request") return false;
          return r.status === "เสนอ" || r.status === "ขอเบิก";
        });
      }
      
      return allRecords.filter(r => 
        r.type === "notification" && 
        r.submitted_by === currentUser.username
      );
    }
    
    function hasUnviewedNotifications() {
      const notifications = getNotifications();
      return notifications.some(n => !viewedNotifications.has(n.id));
    }

    function renderApp() {
      const app = document.getElementById('app');
      
      if (!currentUser) {
        app.innerHTML = renderLogin();
      } else {
        app.innerHTML = renderDashboard();
      }
    }

    function renderLogin() {
      const config = window.elementSdk?.config || defaultConfig;
      const systemTitle = config.system_title || defaultConfig.system_title;
      const orgName = config.organization_name || defaultConfig.organization_name;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const secondaryColor = config.secondary_color || defaultConfig.secondary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      return `
        <div class="w-full min-h-full flex items-center justify-center" style="background-color: ${secondaryColor};">
          <div class="w-full max-w-md mx-4">
            <div class="bg-white rounded-lg shadow-lg p-8">
              <div class="text-center mb-8">
                <h1 class="text-3xl font-bold mb-2" style="color: ${primaryColor};">${systemTitle}</h1>
                <p class="text-lg" style="color: ${textColor};">${orgName}</p>
              </div>
              
              <form id="loginForm" class="space-y-6">
                <div>
                  <label for="username" class="block text-sm font-medium mb-2" style="color: ${textColor};">ชื่อผู้ใช้</label>
                  <input 
                    type="text" 
                    id="username" 
                    required
                    class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                    placeholder="กรอกชื่อผู้ใช้"
                  >
                </div>
                
                <div>
                  <label for="password" class="block text-sm font-medium mb-2" style="color: ${textColor};">รหัสผ่าน</label>
                  <div class="relative">
                    <input 
                      type="password" 
                      id="password" 
                      required
                      class="w-full px-4 py-2 pr-12 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                      placeholder="กรอกรหัสผ่าน"
                    >
                    <button 
                      type="button"
                      onclick="togglePassword()"
                      class="absolute right-3 top-1/2 transform -translate-y-1/2 text-gray-500 hover:text-gray-700"
                    >
                      <svg id="eyeIcon" class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path>
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"></path>
                      </svg>
                    </button>
                  </div>
                </div>
                
                <div id="loginError" class="hidden text-red-600 text-sm"></div>
                
                <button 
                  type="submit" 
                  class="w-full text-white py-3 rounded-lg font-medium hover:opacity-90 transition-opacity"
                  style="background-color: ${primaryColor};"
                >
                  เข้าสู่ระบบ
                </button>
              </form>
              
              <div class="mt-6 pt-6 border-t border-gray-200">
                <p class="text-sm text-center mb-3" style="color: ${textColor};">ตัวอย่างบัญชีผู้ใช้:</p>
                <div class="space-y-2 text-xs" style="color: ${textColor};">
                  <div class="bg-blue-50 p-2 rounded">
                    <strong>เจ้าหน้าที่:</strong> admin / admin123
                  </div>
                  <div class="bg-green-50 p-2 rounded">
                    <strong>หน่วยงาน:</strong> curriculum / dept123
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      `;
    }

    function togglePassword() {
      const passwordInput = document.getElementById('password');
      const eyeIcon = document.getElementById('eyeIcon');
      
      if (passwordInput.type === 'password') {
        passwordInput.type = 'text';
        eyeIcon.innerHTML = `
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13.875 18.825A10.05 10.05 0 0112 19c-4.478 0-8.268-2.943-9.543-7a9.97 9.97 0 011.563-3.029m5.858.908a3 3 0 114.243 4.243M9.878 9.878l4.242 4.242M9.88 9.88l-3.29-3.29m7.532 7.532l3.29 3.29M3 3l3.59 3.59m0 0A9.953 9.953 0 0112 5c4.478 0 8.268 2.943 9.543 7a10.025 10.025 0 01-4.132 5.411m0 0L21 21"></path>
        `;
      } else {
        passwordInput.type = 'password';
        eyeIcon.innerHTML = `
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path>
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"></path>
        `;
      }
    }

    function renderDashboard() {
      const config = window.elementSdk?.config || defaultConfig;
      const systemTitle = config.system_title || defaultConfig.system_title;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const secondaryColor = config.secondary_color || defaultConfig.secondary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const notifications = getNotifications();
      const hasUnviewed = hasUnviewedNotifications();
      
      return `
        <div class="w-full min-h-full" style="background-color: ${secondaryColor};">
          <!-- Header -->
          <header class="bg-white shadow-sm">
            <div class="max-w-7xl mx-auto px-4 py-4 flex justify-between items-center">
              <div>
                <h1 class="text-2xl font-bold" style="color: ${primaryColor};">${systemTitle}</h1>
                <p class="text-sm" style="color: ${textColor};">สวัสดี, ${currentUser.name}</p>
              </div>
              <div class="flex items-center gap-4">
                <button 
                  onclick="toggleNotifications()"
                  class="relative p-2 rounded-lg hover:bg-gray-100 transition-colors"
                  style="color: ${textColor};"
                >
                  <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9"></path>
                  </svg>
                  ${hasUnviewed ? '<span class="notification-dot"></span>' : ''}
                </button>
                <button 
                  onclick="logout()"
                  class="px-4 py-2 text-white rounded-lg hover:opacity-90 transition-opacity"
                  style="background-color: ${primaryColor};"
                >
                  ออกจากระบบ
                </button>
              </div>
            </div>
          </header>

          <!-- Notifications Panel -->
          <div id="notificationsPanel" class="hidden fixed top-16 right-4 w-96 bg-white rounded-lg shadow-xl z-50 max-h-96 overflow-y-auto">
            <div class="p-4 border-b">
              <h3 class="font-semibold" style="color: ${textColor};">การแจ้งเตือน (${notifications.length})</h3>
            </div>
            <div id="notificationsList" class="divide-y">
              ${notifications.length === 0 ? 
                `<div class="p-4 text-center text-gray-500">ไม่มีการแจ้งเตือน</div>` :
                notifications.map(n => `
                  <div class="p-4 hover:bg-gray-50 cursor-pointer" onclick="viewRequestDetail('${n.id}')">
                    <p class="font-medium" style="color: ${textColor};">${n.project_name}</p>
                    <p class="text-sm text-gray-600 mt-1">${n.notes}</p>
                    <p class="text-xs text-gray-400 mt-1">${new Date(n.submitted_date).toLocaleDateString('th-TH')}</p>
                  </div>
                `).join('')
              }
            </div>
          </div>

          <!-- Main Content -->
          <main class="max-w-7xl mx-auto px-4 py-6">
            <div class="mb-6 flex justify-between items-center">
              <div class="flex gap-2">
                ${currentUser.role === "staff" ? `
                  <button 
                    onclick="showView('requests')"
                    id="btnRequests"
                    class="px-4 py-2 rounded-lg font-medium transition-colors"
                    style="background-color: ${primaryColor}; color: white;"
                  >
                    คำ���ออนุ��ัติ
                  </button>
                  <button 
                    onclick="showView('manage')"
                    id="btnManage"
                    class="px-4 py-2 rounded-lg font-medium bg-gray-200 hover:bg-gray-300 transition-colors"
                    style="color: ${textColor};"
                  >
                    จัดการระบบ
                  </button>
                ` : ''}
              </div>
              <button 
                onclick="showAddRequestForm()"
                class="px-6 py-2 text-white rounded-lg font-medium hover:opacity-90 transition-opacity"
                style="background-color: ${primaryColor};"
              >
                + เพิ่มคำขออนุมัติ
              </button>
            </div>

            <div id="mainContent">
              ${renderRequestsList()}
            </div>
          </main>
        </div>
      `;
    }

    function renderRequestsList() {
      const config = window.elementSdk?.config || defaultConfig;
      const textColor = config.text_color || defaultConfig.text_color;
      
      let filteredRecords = allRecords.filter(r => r.type === "request");
      
      if (currentUser.role === "department") {
        filteredRecords = filteredRecords.filter(r => r.submitted_by === currentUser.username);
      }
      
      if (filteredRecords.length === 0) {
        return `
          <div class="bg-white rounded-lg shadow p-8 text-center">
            <svg class="w-16 h-16 mx-auto mb-4 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"></path>
            </svg>
            <p class="text-lg" style="color: ${textColor};">ยังไม่มีคำขออนุมัต���</p>
            <p class="text-sm text-gray-500 mt-2">เริ่มต้นโดยการเพิ่มคำขออนุมัติใหม่</p>
          </div>
        `;
      }
      
      return `
        <div class="bg-white rounded-lg shadow overflow-hidden">
          <div class="overflow-x-auto">
            <table class="w-full">
              <thead style="background-color: ${config.secondary_color || defaultConfig.secondary_color};">
                <tr>
                  <th class="px-4 py-3 text-left text-sm font-semibold" style="color: ${textColor};">เลขที่เอกสาร</th>
                  <th class="px-4 py-3 text-left text-sm font-semibold" style="color: ${textColor};">ชื่อโครงการ</th>
                  <th class="px-4 py-3 text-left text-sm font-semibold" style="color: ${textColor};">หน่วยงาน</th>
                  <th class="px-4 py-3 text-left text-sm font-semibold" style="color: ${textColor};">จำนวนเงิน</th>
                  <th class="px-4 py-3 text-left text-sm font-semibold" style="color: ${textColor};">สถานะ</th>
                  <th class="px-4 py-3 text-left text-sm font-semibold" style="color: ${textColor};">วันที่</th>
                  <th class="px-4 py-3 text-center text-sm font-semibold" style="color: ${textColor};">จัดการ</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-gray-200">
                ${filteredRecords.map(record => `
                  <tr class="hover:bg-gray-50" data-record-id="${record.id}">
                    <td class="px-4 py-3 text-sm" style="color: ${textColor};">${record.document_number}</td>
                    <td class="px-4 py-3 text-sm" style="color: ${textColor};">${record.project_name}</td>
                    <td class="px-4 py-3 text-sm" style="color: ${textColor};">${record.department}</td>
                    <td class="px-4 py-3 text-sm" style="color: ${textColor};">${parseFloat(record.budget_amount).toLocaleString()} บาท</td>
                    <td class="px-4 py-3">
                      <span class="status-badge" style="background-color: ${getStatusColor(record.status)}20; color: ${getStatusColor(record.status)};">
                        ${record.status}
                      </span>
                    </td>
                    <td class="px-4 py-3 text-sm" style="color: ${textColor};">${new Date(record.submitted_date).toLocaleDateString('th-TH')}</td>
                    <td class="px-4 py-3 text-center">
                      <button 
                        onclick="viewRequestDetail('${record.id}')"
                        class="text-blue-600 hover:text-blue-800 mr-2"
                      >
                        ���ู
                      </button>
                      ${currentUser.role === "staff" || record.submitted_by === currentUser.username ? `
                        <button 
                          onclick="editRequest('${record.id}')"
                          class="text-green-600 hover:text-green-800 mr-2"
                        >
                          แก้ไข
                        </button>
                        <button 
                          onclick="confirmDeleteRequest('${record.id}')"
                          class="text-red-600 hover:text-red-800"
                        >
                          ลบ
                        </button>
                      ` : ''}
                    </td>
                  </tr>
                `).join('')}
              </tbody>
            </table>
          </div>
        </div>
      `;
    }

    function renderManageView() {
      const config = window.elementSdk?.config || defaultConfig;
      const textColor = config.text_color || defaultConfig.text_color;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      
      return `
        <div class="bg-white rounded-lg shadow p-6">
          <h2 class="text-xl font-bold mb-6" style="color: ${textColor};">จ���ดการระบบ</h2>
          
          <div class="mb-8">
            <div class="flex justify-between items-center mb-4">
              <h3 class="text-lg font-semibold" style="color: ${textColor};">เจ้าหน้าที่</h3>
            </div>
            <div class="space-y-2">
              ${systemUsers.staff.map(user => `
                <div class="flex justify-between items-center p-3 bg-blue-50 rounded-lg">
                  <div>
                    <p class="font-medium" style="color: ${textColor};">${user.name}</p>
                    <p class="text-sm text-gray-600">ชื่อผู้ใช้: ${user.username} | ตำแหน่ง: ${user.position}</p>
                  </div>
                  <span class="px-3 py-1 bg-blue-500 text-white text-sm rounded-full">เจ้าหน้าที่</span>
                </div>
              `).join('')}
            </div>
          </div>
          
          <div>
            <div class="flex justify-between items-center mb-4">
              <h3 class="text-lg font-semibold" style="color: ${textColor};">ผู้ใช้ตามหน่วยงาน</h3>
              <button 
                onclick="showAddDepartmentUserForm()"
                class="px-4 py-2 text-white rounded-lg text-sm hover:opacity-90"
                style="background-color: ${primaryColor};"
              >
                + เพิ่มผู้ใช้
              </button>
            </div>
            <div class="space-y-2">
              ${systemUsers.departments.map(user => `
                <div class="flex justify-between items-center p-3 bg-green-50 rounded-lg">
                  <div>
                    <p class="font-medium" style="color: ${textColor};">${user.name}</p>
                    <p class="text-sm text-gray-600">ชื่อผู้ใช้: ${user.username} | หน่ว��งาน: ${user.department}</p>
                  </div>
                  <div class="flex gap-2">
                    <button 
                      onclick="editDepartmentUser('${user.id}')"
                      class="px-3 py-1 bg-yellow-500 text-white text-sm rounded hover:bg-yellow-600"
                    >
                      แก้ไข
                    </button>
                    <button 
                      onclick="confirmDeleteDepartmentUser('${user.id}')"
                      class="px-3 py-1 bg-red-500 text-white text-sm rounded hover:bg-red-600"
                    >
                      ลบ
                    </button>
                  </div>
                </div>
              `).join('')}
            </div>
          </div>
        </div>
      `;
    }

    async function handleLogin(event) {
      event.preventDefault();
      
      const username = document.getElementById('username').value;
      const password = document.getElementById('password').value;
      const errorDiv = document.getElementById('loginError');
      
      const allUsers = [...systemUsers.staff, ...systemUsers.departments];
      const user = allUsers.find(u => u.username === username && u.password === password);
      
      if (user) {
        currentUser = user;
        renderApp();
      } else {
        errorDiv.textContent = "ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง";
        errorDiv.classList.remove('hidden');
      }
    }

    function logout() {
      currentUser = null;
      renderApp();
    }

    function toggleNotifications() {
      showingNotifications = !showingNotifications;
      const panel = document.getElementById('notificationsPanel');
      if (showingNotifications) {
        panel.classList.remove('hidden');
        const notifications = getNotifications();
        notifications.forEach(n => {
          viewedNotifications.add(n.id);
        });
        renderApp();
      } else {
        panel.classList.add('hidden');
      }
    }

    function showView(view) {
      const mainContent = document.getElementById('mainContent');
      const btnRequests = document.getElementById('btnRequests');
      const btnManage = document.getElementById('btnManage');
      
      const config = window.elementSdk?.config || defaultConfig;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      if (view === 'requests') {
        mainContent.innerHTML = renderRequestsList();
        btnRequests.style.backgroundColor = primaryColor;
        btnRequests.style.color = 'white';
        btnManage.style.backgroundColor = '#e5e7eb';
        btnManage.style.color = textColor;
      } else if (view === 'manage') {
        mainContent.innerHTML = renderManageView();
        btnManage.style.backgroundColor = primaryColor;
        btnManage.style.color = 'white';
        btnRequests.style.backgroundColor = '#e5e7eb';
        btnRequests.style.color = textColor;
      }
    }

    function showAddRequestForm() {
      const config = window.elementSdk?.config || defaultConfig;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const autoDocNumber = generateDocumentNumber();
      
      const modal = document.createElement('div');
      modal.className = 'fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4';
      modal.innerHTML = `
        <div class="bg-white rounded-lg shadow-xl w-full max-w-2xl max-h-full overflow-y-auto">
          <div class="p-6 border-b flex justify-between items-center">
            <h2 class="text-xl font-bold" style="color: ${textColor};">เพิ่มคำขออนุมัติใหม่</h2>
            <button onclick="this.closest('.fixed').remove()" class="text-gray-500 hover:text-gray-700">
              <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
              </svg>
            </button>
          </div>
          <form id="addRequestForm" class="p-6 space-y-4">
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">เลขที��เอกสาร (สร้างอัตโนมัติ)</label>
              <input type="text" name="document_number" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 bg-gray-50" value="${autoDocNumber}" readonly>
              <p class="text-xs text-gray-500 mt-1">เลขที่เอกสารจะถูกสร้างอัตโนมัติ</p>
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">ชื่อโครงการ</label>
              <input type="text" name="project_name" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="ระบุชื่อโครงการ">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">หน่วยงาน</label>
              <input type="text" name="department" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="ระบุชื่อหน่วยงาน" value="${currentUser.department || ''}">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">จำนวนงบประมาณ (บาท)</label>
              <input type="number" name="budget_amount" required min="0" step="0.01" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="0.00">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">หมายเหตุ</label>
              <textarea name="notes" rows="3" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="ระบุรายละเอียดเพิ่มเติม"></textarea>
            </div>
            <div class="flex justify-end gap-3 pt-4">
              <button type="button" onclick="this.closest('.fixed').remove()" class="px-6 py-2 border border-gray-300 rounded-lg hover:bg-gray-50" style="color: ${textColor};">
                ยก������ิก
              </button>
              <button type="submit" class="px-6 py-2 text-white rounded-lg hover:opacity-90" style="background-color: ${primaryColor};">
                บันทึก
              </button>
            </div>
          </form>
        </div>
      `;
      
      document.body.appendChild(modal);
      
      document.getElementById('addRequestForm').addEventListener('submit', handleAddRequest);
    }

    async function handleAddRequest(event) {
      event.preventDefault();
      
      const formData = new FormData(event.target);
      const submitButton = event.target.querySelector('button[type="submit"]');
      submitButton.disabled = true;
      submitButton.textContent = 'กำลังบันทึก...';
      
      const newRequest = {
        id: 'REQ-' + Date.now(),
        document_number: formData.get('document_number'),
        project_name: formData.get('project_name'),
        department: formData.get('department'),
        budget_amount: parseFloat(formData.get('budget_amount')),
        status: 'เสนอ',
        submitted_by: currentUser.username,
        submitted_date: new Date().toISOString(),
        current_approver: 'staff',
        notes: formData.get('notes') || '',
        history: JSON.stringify([{
          status: 'เสนอ',
          date: new Date().toISOString(),
          user: currentUser.name,
          action: 'สร้างคำขออนุมัติ'
        }]),
        type: 'request'
      };
      
      if (allRecords.length >= 999) {
        submitButton.disabled = false;
        submitButton.textContent = 'บันทึก';
        
        const errorDiv = document.createElement('div');
        errorDiv.className = 'text-red-600 text-sm mt-2';
        errorDiv.textContent = 'ถึงขีดจำกัดจำนวนเอกสาร 999 รายการแล้ว กรุณาลบเอกสารเก่าก่อนเ�����่ม��หม่';
        event.target.appendChild(errorDiv);
        return;
      }
      
      const result = await window.dataSdk.create(newRequest);
      
      if (result.isOk) {
        event.target.closest('.fixed').remove();
      } else {
        submitButton.disabled = false;
        submitButton.textContent = 'บันทึก';
        const errorDiv = document.createElement('div');
        errorDiv.className = 'text-red-600 text-sm mt-2';
        errorDiv.textContent = 'เกิดข้อผิดพลาดในการบันทึกข้อมูล';
        event.target.appendChild(errorDiv);
      }
    }

    function viewRequestDetail(id) {
      const record = allRecords.find(r => r.id === id && r.type === "request");
      if (!record) return;
      
      const config = window.elementSdk?.config || defaultConfig;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const history = JSON.parse(record.history || '[]');
      
      const canApprove = currentUser.role === "staff" && record.status === "เสนอ";
      const canRequestDisbursement = (currentUser.role === "department" || currentUser.role === "staff") && record.status === "ดำเนินกิจกรรม" && (record.submitted_by === currentUser.username || currentUser.role === "staff");
      const canProposeToManager = currentUser.role === "staff" && record.status === "ขอเบิก";
      const canApprovePayment = currentUser.role === "staff" && record.status === "เสนอผู้บริหาร";
      const canCompletePayment = currentUser.role === "staff" && record.status === "���นุมัติเบิก";
      const canProposeAfterEdit = (currentUser.role === "department" || currentUser.role === "staff") && record.status === "รอแก้ไข" && (record.submitted_by === currentUser.username || currentUser.role === "staff");
      const canRequestAfterEdit = (currentUser.role === "department" || currentUser.role === "staff") && record.status === "รอแก้ไข-ตั้งเบิก" && (record.submitted_by === currentUser.username || currentUser.role === "staff");
      const canReturnDocument = currentUser.role === "staff" && (record.status === "เสนอ" || record.status === "ดำเนินกิจกรรม" || record.status === "ขอเบิก" || record.status === "เสนอผู้บริหาร" || record.status === "อนุมัติเบิก");
      
      const modal = document.createElement('div');
      modal.className = 'fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4';
      modal.innerHTML = `
        <div class="bg-white rounded-lg shadow-xl w-full max-w-3xl flex flex-col" style="max-height: 90vh;">
          <div class="p-6 border-b flex justify-between items-center flex-shrink-0 bg-white rounded-t-lg">
            <h2 class="text-xl font-bold" style="color: ${textColor};">รายละเอียดคำขออนุมัติ</h2>
            <button id="closeDetailModal" class="text-gray-500 hover:text-gray-700">
              <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
              </svg>
            </button>
          </div>
          <div class="p-6 overflow-y-auto flex-1">
            <div class="grid grid-cols-2 gap-4 mb-6">
              <div>
                <p class="text-sm text-gray-600">เลขที่เอกสาร</p>
                <p class="font-medium" style="color: ${textColor};">${record.document_number}</p>
              </div>
              <div>
                <p class="text-sm text-gray-600">สถานะ</p>
                <span class="status-badge" style="background-color: ${getStatusColor(record.status)}20; color: ${getStatusColor(record.status)};">
                  ${record.status}
                </span>
              </div>
              <div>
                <p class="text-sm text-gray-600">ชื่อโครงการ</p>
                <p class="font-medium" style="color: ${textColor};">${record.project_name}</p>
              </div>
              <div>
                <p class="text-sm text-gray-600">��น่วยงาน</p>
                <p class="font-medium" style="color: ${textColor};">${record.department}</p>
              </div>
              <div>
                <p class="text-sm text-gray-600">จำนวนเงิน</p>
                <p class="font-medium" style="color: ${textColor};">${parseFloat(record.budget_amount).toLocaleString()} บาท</p>
              </div>
              <div>
                <p class="text-sm text-gray-600">วันที่ส่ง</p>
                <p class="font-medium" style="color: ${textColor};">${new Date(record.submitted_date).toLocaleDateString('th-TH')}</p>
              </div>
            </div>
            
            ${record.notes ? `
              <div class="mb-6">
                <p class="text-sm text-gray-600 mb-2">หมายเหตุ</p>
                <p style="color: ${textColor};">${record.notes}</p>
              </div>
            ` : ''}
            
            <div class="mb-6">
              <h3 class="font-semibold mb-3" style="color: ${textColor};">ประวัติการดำเนินการ</h3>
              <div class="space-y-3 max-h-64 overflow-y-auto border rounded-lg p-3">
                ${history.map(h => `
                  <div class="flex gap-3 p-3 bg-gray-50 rounded-lg">
                    <div class="flex-shrink-0 w-3 h-3 rounded-full mt-1" style="background-color: ${getStatusColor(h.status)};"></div>
                    <div class="flex-1">
                      <p class="font-medium" style="color: ${textColor};">${h.status}</p>
                      <p class="text-sm text-gray-600">${h.action} โดย ${h.user}</p>
                      <p class="text-xs text-gray-400">${new Date(h.date).toLocaleString('th-TH')}</p>
                    </div>
                  </div>
                `).join('')}
              </div>
            </div>
          </div>
          
          <div class="border-t p-6 flex-shrink-0 bg-white rounded-b-lg">
            <div class="space-y-3">
              ${canApprove ? `
                <button 
                  id="btnApprove"
                  class="px-6 py-3 text-white rounded-lg font-medium hover:opacity-90 w-full"
                  style="background-color: ${config.success_color || defaultConfig.success_color};"
                >
                  อนุมัติงบประมาณ
                </button>
              ` : ''}
              
              ${canRequestDisbursement ? `
                <button 
                  id="btnRequestDisbursement"
                  class="px-6 py-3 text-white rounded-lg font-medium hover:opacity-90 w-full"
                  style="background-color: ${primaryColor};"
                >
                  ขอเบิกเงิน
                </button>
              ` : ''}
              
              ${canProposeToManager ? `
                <button 
                  id="btnProposeToManager"
                  class="px-6 py-3 rounded-lg font-medium hover:opacity-80 w-full"
                  style="background-color: ${getStatusColor('เสนอผู้บริหาร')}; color: white;"
                >
                  เสนอผู้บริหาร
                </button>
              ` : ''}
              
              ${canApprovePayment ? `
                <button 
                  id="btnApprovePayment"
                  class="px-6 py-3 rounded-lg font-medium hover:opacity-80 w-full"
                  style="background-color: ${getStatusColor('อนุมัติเบิก')}; color: white;"
                >
                  อนุมัติเบิก
                </button>
              ` : ''}
              
              ${canCompletePayment ? `
                <button 
                  id="btnCompletePayment"
                  class="px-6 py-3 rounded-lg font-medium hover:opacity-80 w-full"
                  style="background-color: ${getStatusColor('จ่ายเงินเสร็จสิ้น')}; color: white;"
                >
                  จ่ายเงินเสร็จสิ้น
                </button>
              ` : ''}
              
              ${canProposeAfterEdit ? `
                <button 
                  id="btnProposeAfterEdit"
                  class="px-6 py-3 text-white rounded-lg font-medium hover:opacity-90 w-full"
                  style="background-color: ${getStatusColor('เสนอ')};"
                >
                  เสนอกิจกรรม
                </button>
              ` : ''}
              
              ${canRequestAfterEdit ? `
                <button 
                  id="btnRequestAfterEdit"
                  class="px-6 py-3 text-white rounded-lg font-medium hover:opacity-90 w-full"
                  style="background-color: ${getStatusColor('ขอเบิก')};"
                >
                  ส่งคำขอเบิกเงิน
                </button>
              ` : ''}
              
              ${canReturnDocument ? `
                <button 
                  id="btnReturnDocument"
                  class="px-6 py-3 bg-orange-500 text-white rounded-lg font-medium hover:opacity-90 w-full"
                >
                  ส่งเอกสารคืน
                </button>
              ` : ''}
            </div>
          </div>
        </div>
      `;
      
      document.body.appendChild(modal);
      
      // เพิ่ม Event Listeners หลังจาก modal ถูกเพิ่มเข้า DOM
      const closeBtn = modal.querySelector('#closeDetailModal');
      if (closeBtn) {
        closeBtn.addEventListener('click', () => modal.remove());
      }
      
      if (canApprove) {
        const btn = modal.querySelector('#btnApprove');
        if (btn) btn.addEventListener('click', () => approveRequest(record.id));
      }
      
      if (canRequestDisbursement) {
        const btn = modal.querySelector('#btnRequestDisbursement');
        if (btn) btn.addEventListener('click', () => requestDisbursement(record.id));
      }
      
      if (canProposeToManager) {
        const btn = modal.querySelector('#btnProposeToManager');
        if (btn) btn.addEventListener('click', () => updateRequestStatus(record.id, 'เสนอผู้บ���ิหาร'));
      }
      
      if (canApprovePayment) {
        const btn = modal.querySelector('#btnApprovePayment');
        if (btn) btn.addEventListener('click', () => updateRequestStatus(record.id, 'อนุมัติเบิก'));
      }
      
      if (canCompletePayment) {
        const btn = modal.querySelector('#btnCompletePayment');
        if (btn) btn.addEventListener('click', () => updateRequestStatus(record.id, 'จ่ายเงินเสร็จสิ้น'));
      }
      
      if (canProposeAfterEdit) {
        const btn = modal.querySelector('#btnProposeAfterEdit');
        if (btn) btn.addEventListener('click', () => proposeAfterEdit(record.id));
      }
      
      if (canRequestAfterEdit) {
        const btn = modal.querySelector('#btnRequestAfterEdit');
        if (btn) btn.addEventListener('click', () => requestAfterEdit(record.id));
      }
      
      if (canReturnDocument) {
        const btn = modal.querySelector('#btnReturnDocument');
        if (btn) btn.addEventListener('click', () => showReturnDocumentForm(record.id));
      }
    }

    async function approveRequest(id) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const history = JSON.parse(record.history || '[]');
      history.push({
        status: '���ำเนินกิจกรรม',
        date: new Date().toISOString(),
        user: currentUser.name,
        action: 'อนุมัติงบประมาณ'
      });
      
      const updatedRecord = {
        ...record,
        status: '��ำเนินกิจกรรม',
        history: JSON.stringify(history)
      };
      
      const result = await window.dataSdk.update(updatedRecord);
      
      if (result.isOk) {
        document.querySelectorAll('.fixed').forEach(m => m.remove());
        
        const notification = {
          id: 'NOTIF-' + Date.now(),
          type: 'notification',
          submitted_by: record.submitted_by,
          project_name: record.project_name,
          document_number: record.document_number,
          notes: `งบประมาณได้รับการอนุมัติแล้ว สามารถดำเนินกิจกรรมได้`,
          submitted_date: new Date().toISOString(),
          status: 'ดำเนินกิจกรรม',
          department: record.department,
          budget_amount: record.budget_amount,
          current_approver: '',
          history: ''
        };
        
        await window.dataSdk.create(notification);
      }
    }

    async function requestDisbursement(id) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const history = JSON.parse(record.history || '[]');
      history.push({
        status: 'ขอเบิก',
        date: new Date().toISOString(),
        user: currentUser.name,
        action: '��่งคำขอเบิกเงิน'
      });
      
      const updatedRecord = {
        ...record,
        status: 'ขอเบิก',
        history: JSON.stringify(history)
      };
      
      const result = await window.dataSdk.update(updatedRecord);
      
      if (result.isOk) {
        document.querySelectorAll('.fixed').forEach(m => m.remove());
      }
    }

    async function updateRequestStatus(id, newStatus) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const history = JSON.parse(record.history || '[]');
      history.push({
        status: newStatus,
        date: new Date().toISOString(),
        user: currentUser.name,
        action: 'เปลี่ยนสถานะเป็น ' + newStatus
      });
      
      const updatedRecord = {
        ...record,
        status: newStatus,
        history: JSON.stringify(history)
      };
      
      const result = await window.dataSdk.update(updatedRecord);
      
      if (result.isOk) {
        document.querySelectorAll('.fixed').forEach(m => m.remove());
        
        const notification = {
          id: 'NOTIF-' + Date.now(),
          type: 'notification',
          submitted_by: record.submitted_by,
          project_name: record.project_name,
          document_number: record.document_number,
          notes: `สถานะเอกสารถูกเปลี่ยนเป็น "${newStatus}"`,
          submitted_date: new Date().toISOString(),
          status: newStatus,
          department: record.department,
          budget_amount: record.budget_amount,
          current_approver: '',
          history: ''
        };
        
        await window.dataSdk.create(notification);
      }
    }

    async function proposeAfterEdit(id) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const history = JSON.parse(record.history || '[]');
      history.push({
        status: 'เสนอ',
        date: new Date().toISOString(),
        user: currentUser.name,
        action: 'เสนอกิจกรร���หลังแก้ไข'
      });
      
      const updatedRecord = {
        ...record,
        status: 'เสนอ',
        history: JSON.stringify(history)
      };
      
      const result = await window.dataSdk.update(updatedRecord);
      
      if (result.isOk) {
        document.querySelectorAll('.fixed').forEach(m => m.remove());
      }
    }

    async function requestAfterEdit(id) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const history = JSON.parse(record.history || '[]');
      history.push({
        status: 'ขอเบิก',
        date: new Date().toISOString(),
        user: currentUser.name,
        action: 'ขอเบิกเงินหลังแก้ไข'
      });
      
      const updatedRecord = {
        ...record,
        status: 'ขอเบิก',
        history: JSON.stringify(history)
      };
      
      const result = await window.dataSdk.update(updatedRecord);
      
      if (result.isOk) {
        document.querySelectorAll('.fixed').forEach(m => m.remove());
      }
    }

    function showReturnDocumentForm(id) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const config = window.elementSdk?.config || defaultConfig;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const modal = document.createElement('div');
      modal.className = 'fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4';
      modal.style.zIndex = '60';
      modal.innerHTML = `
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md p-6">
          <h3 class="text-lg font-bold mb-4" style="color: ${textColor};">ส่งเอกสารคืน</h3>
          <form id="returnForm">
            <div class="mb-4">
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">เหตุผลในการส่งคืน</label>
              <textarea 
                name="reason" 
                required 
                rows="4" 
                class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                placeholder="ระบุเหตุผล..."
              ></textarea>
            </div>
            <div class="flex justify-end gap-3">
              <button type="button" id="btnCancelReturn" class="px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50">
                ยกเลิก
              </button>
              <button type="submit" class="px-4 py-2 bg-orange-500 text-white rounded-lg hover:bg-orange-600">
                ส่งคืน
              </button>
            </div>
          </form>
        </div>
      `;
      
      document.body.appendChild(modal);
      
      modal.querySelector('#btnCancelReturn').addEventListener('click', () => modal.remove());
      
      modal.querySelector('#returnForm').addEventListener('submit', async (e) => {
        e.preventDefault();
        const submitButton = e.target.querySelector('button[type="submit"]');
        submitButton.disabled = true;
        submitButton.textContent = 'กำลังส่งคืน...';
        
        const reason = e.target.reason.value;
        
        let newStatus = 'รอแก้ไข';
        
        const statusesAfterDisbursement = ['ขอเบิก', 'เสนอผู้บริหาร', 'อนุมัติเบิก'];
        if (statusesAfterDisbursement.includes(record.status)) {
          newStatus = 'รอแก้ไข-ตั้งเบิก';
        }
        
        // ต้องห��ข้อม���ลล่าสุดจาก allRecords เพราะ record ที่ส่งเข้ามาอาจเก่า
        const currentRecord = allRecords.find(r => r.id === record.id);
        if (!currentRecord) {
          submitButton.disabled = false;
          submitButton.textContent = 'ส่งคืน';
          const errorDiv = document.createElement('div');
          errorDiv.className = 'text-red-600 text-sm mt-2';
          errorDiv.textContent = 'ไม่พบข้อมูลเอกสาร';
          e.target.appendChild(errorDiv);
          return;
        }
        
        const history = JSON.parse(currentRecord.history || '[]');
        history.push({
          status: newStatus,
          date: new Date().toISOString(),
          user: currentUser.name,
          action: 'ส่งเอกสารคืน - ' + reason
        });
        
        const updatedRecord = {
          ...currentRecord,
          status: newStatus,
          history: JSON.stringify(history)
        };
        
        const result = await window.dataSdk.update(updatedRecord);
        
        if (result.isOk) {
          const notification = {
            id: 'NOTIF-' + Date.now(),
            type: 'notification',
            submitted_by: currentRecord.submitted_by,
            project_name: currentRecord.project_name,
            document_number: currentRecord.document_number,
            notes: `เอกสารถูกส่งคืน: ${reason}`,
            submitted_date: new Date().toISOString(),
            status: 'ส่งคืน',
            department: currentRecord.department,
            budget_amount: currentRecord.budget_amount,
            current_approver: '',
            history: ''
          };
          
          await window.dataSdk.create(notification);
          
          document.querySelectorAll('.fixed').forEach(m => m.remove());
        } else {
          submitButton.disabled = false;
          submitButton.textContent = 'ส่งคืน';
          
          const errorDiv = document.createElement('div');
          errorDiv.className = 'text-red-600 text-sm mt-2';
          errorDiv.textContent = 'เกิดข้อผิดพลาดในการส่งคืนเอกสาร: ' + (result.error?.message || 'ไม่ทราบสาเหตุ');
          e.target.appendChild(errorDiv);
        }
      });
    }

    function editRequest(id) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const config = window.elementSdk?.config || defaultConfig;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const modal = document.createElement('div');
      modal.className = 'fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4';
      modal.innerHTML = `
        <div class="bg-white rounded-lg shadow-xl w-full max-w-2xl max-h-full overflow-y-auto">
          <div class="p-6 border-b flex justify-between items-center">
            <h2 class="text-xl font-bold" style="color: ${textColor};">แก้ไขคำขออนุมัติ</h2>
            <button onclick="this.closest('.fixed').remove()" class="text-gray-500 hover:text-gray-700">
              <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
              </svg>
            </button>
          </div>
          <form id="editRequestForm" class="p-6 space-y-4">
            <input type="hidden" name="id" value="${record.id}">
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">เลขที่เอกสาร</label>
              <input type="text" name="document_number" required value="${record.document_number}" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" ${currentUser.role === "department" ? 'readonly' : ''}>
              ${currentUser.role === "staff" ? '<p class="text-xs text-gray-500 mt-1">เจ้าหน้าที่สามารถแก้ไขเลขที่เอกสารได้</p>' : ''}
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">ชื่อโครงการ</label>
              <input type="text" name="project_name" required value="${record.project_name}" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">หน่วยงาน</label>
              <input type="text" name="department" required value="${record.department}" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">จำนวนงบประมาณ (บาท)</label>
              <input type="number" name="budget_amount" required min="0" step="0.01" value="${record.budget_amount}" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">หมายเหตุ</label>
              <textarea name="notes" rows="3" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">${record.notes}</textarea>
            </div>
            <div class="flex justify-end gap-3 pt-4">
              <button type="button" onclick="this.closest('.fixed').remove()" class="px-6 py-2 border border-gray-300 rounded-lg hover:bg-gray-50" style="color: ${textColor};">
                ยกเลิก
              </button>
              <button type="submit" class="px-6 py-2 text-white rounded-lg hover:opacity-90" style="background-color: ${primaryColor};">
                บันทึกการแก้ไข
              </button>
            </div>
          </form>
        </div>
      `;
      
      document.body.appendChild(modal);
      
      document.getElementById('editRequestForm').addEventListener('submit', handleEditRequest);
    }

    async function handleEditRequest(event) {
      event.preventDefault();
      
      const formData = new FormData(event.target);
      const id = formData.get('id');
      const record = allRecords.find(r => r.id === id);
      
      const submitButton = event.target.querySelector('button[type="submit"]');
      submitButton.disabled = true;
      submitButton.textContent = 'กำลังบันทึก...';
      
      const history = JSON.parse(record.history || '[]');
      history.push({
        status: record.status,
        date: new Date().toISOString(),
        user: currentUser.name,
        action: 'แก้ไข��้อมูล'
      });
      
      const updatedRecord = {
        ...record,
        document_number: formData.get('document_number'),
        project_name: formData.get('project_name'),
        department: formData.get('department'),
        budget_amount: parseFloat(formData.get('budget_amount')),
        notes: formData.get('notes'),
        history: JSON.stringify(history)
      };
      
      const result = await window.dataSdk.update(updatedRecord);
      
      if (result.isOk) {
        event.target.closest('.fixed').remove();
      } else {
        submitButton.disabled = false;
        submitButton.textContent = 'บันทึกการแก้ไข';
        const errorDiv = document.createElement('div');
        errorDiv.className = 'text-red-600 text-sm mt-2';
        errorDiv.textContent = 'เกิดข้อผิดพลาดในการบันทึกข้อมูล';
        event.target.appendChild(errorDiv);
      }
    }

    function confirmDeleteRequest(id) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const config = window.elementSdk?.config || defaultConfig;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const modal = document.createElement('div');
      modal.className = 'fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4';
      modal.innerHTML = `
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md p-6">
          <h3 class="text-lg font-bold mb-4" style="color: ${textColor};">ยืนยันการลบ</h3>
          <p class="mb-6" style="color: ${textColor};">คุณแน่ใจหรือไม่ว่าต้องการลบเอกสาร "${record.document_number}"?</p>
          <div class="flex justify-end gap-3">
            <button onclick="this.closest('.fixed').remove()" class="px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50">
              ยก��ลิก
            </button>
            <button onclick="deleteRequest('${id}')" class="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700">
              ลบ
            </button>
          </div>
        </div>
      `;
      
      document.body.appendChild(modal);
    }

    async function deleteRequest(id) {
      const record = allRecords.find(r => r.id === id);
      if (!record) return;
      
      const result = await window.dataSdk.delete(record);
      
      if (result.isOk) {
        document.querySelector('.fixed')?.remove();
      }
    }

    function showAddDepartmentUserForm() {
      const config = window.elementSdk?.config || defaultConfig;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const modal = document.createElement('div');
      modal.className = 'fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4';
      modal.innerHTML = `
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md p-6">
          <h3 class="text-lg font-bold mb-4" style="color: ${textColor};">เพิ่มผู้ใช้ตามหน่วยงาน</h3>
          <form id="addUserForm" class="space-y-4">
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">ชื่อผู้ใช้</label>
              <input type="text" name="username" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="username">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">รหัสผ่าน</label>
              <input type="password" name="password" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="password">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">ชื่อ-นามสกุล</label>
              <input type="text" name="name" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="ชื่อเต็ม">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">หน่วยงาน</label>
              <input type="text" name="department" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="ชื่อหน่วยงาน">
            </div>
            <div class="flex justify-end gap-3 pt-4">
              <button type="button" onclick="this.closest('.fixed').remove()" class="px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50">
                ยกเ���ิก
              </button>
              <button type="submit" class="px-4 py-2 text-white rounded-lg hover:opacity-90" style="background-color: ${primaryColor};">
                เพิ่มผู้ใช้
              </button>
            </div>
          </form>
        </div>
      `;
      
      document.body.appendChild(modal);
      
      modal.querySelector('#addUserForm').addEventListener('submit', (e) => {
        e.preventDefault();
        const formData = new FormData(e.target);
        
        const newUser = {
          id: 'dept-' + Date.now(),
          username: formData.get('username'),
          password: formData.get('password'),
          name: formData.get('name'),
          department: formData.get('department'),
          role: 'department'
        };
        
        systemUsers.departments.push(newUser);
        modal.remove();
        
        const mainContent = document.getElementById('mainContent');
        mainContent.innerHTML = renderManageView();
      });
    }

    function editDepartmentUser(userId) {
      const user = systemUsers.departments.find(u => u.id === userId);
      if (!user) return;
      
      const config = window.elementSdk?.config || defaultConfig;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const modal = document.createElement('div');
      modal.className = 'fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4';
      modal.innerHTML = `
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md p-6">
          <h3 class="text-lg font-bold mb-4" style="color: ${textColor};">แก้ไขผู้ใช้</h3>
          <form id="editUserForm" class="space-y-4">
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">ชื่อผู้ใช้</label>
              <input type="text" name="username" required value="${user.username}" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">รหัสผ่านใหม่</label>
              <input type="password" name="password" placeholder="เว้นว่างหากไม่ต้องการเปลี่ยน" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">ชื่อ-นามสกุล</label>
              <input type="text" name="name" required value="${user.name}" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
            </div>
            <div>
              <label class="block text-sm font-medium mb-2" style="color: ${textColor};">หน่วยงาน</label>
              <input type="text" name="department" required value="${user.department}" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
            </div>
            <div class="flex justify-end gap-3 pt-4">
              <button type="button" onclick="this.closest('.fixed').remove()" class="px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50">
                ยกเลิก
              </button>
              <button type="submit" class="px-4 py-2 text-white rounded-lg hover:opacity-90" style="background-color: ${primaryColor};">
                บันทึก
              </button>
            </div>
          </form>
        </div>
      `;
      
      document.body.appendChild(modal);
      
      modal.querySelector('#editUserForm').addEventListener('submit', (e) => {
        e.preventDefault();
        const formData = new FormData(e.target);
        
        user.username = formData.get('username');
        if (formData.get('password')) {
          user.password = formData.get('password');
        }
        user.name = formData.get('name');
        user.department = formData.get('department');
        
        modal.remove();
        
        const mainContent = document.getElementById('mainContent');
        mainContent.innerHTML = renderManageView();
      });
    }

    function confirmDeleteDepartmentUser(userId) {
      const user = systemUsers.departments.find(u => u.id === userId);
      if (!user) return;
      
      const config = window.elementSdk?.config || defaultConfig;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const modal = document.createElement('div');
      modal.className = 'fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4';
      modal.innerHTML = `
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md p-6">
          <h3 class="text-lg font-bold mb-4" style="color: ${textColor};">ยืนยันการลบ</h3>
          <p class="mb-6" style="color: ${textColor};">คุณแน่ใจหรือไม่ว่าต้องการลบผู้ใช้ "${user.name}"?</p>
          <div class="flex justify-end gap-3">
            <button onclick="this.closest('.fixed').remove()" class="px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50">
              ยกเลิก
            </button>
            <button onclick="deleteDepartmentUser('${userId}')" class="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700">
              ลบ
            </button>
          </div>
        </div>
      `;
      
      document.body.appendChild(modal);
    }

    function deleteDepartmentUser(userId) {
      const index = systemUsers.departments.findIndex(u => u.id === userId);
      if (index !== -1) {
        systemUsers.departments.splice(index, 1);
        document.querySelector('.fixed')?.remove();
        
        const mainContent = document.getElementById('mainContent');
        mainContent.innerHTML = renderManageView();
      }
    }

    const dataHandler = {
      onDataChanged(data) {
        allRecords = data;
        
        if (currentUser) {
          const mainContent = document.getElementById('mainContent');
          if (mainContent && !mainContent.querySelector('form')) {
            const currentView = document.getElementById('btnManage')?.style.backgroundColor;
            const isManageView = currentView && currentView !== 'rgb(229, 231, 235)';
            
            if (!isManageView) {
              mainContent.innerHTML = renderRequestsList();
            }
          }
          
          const notificationsList = document.getElementById('notificationsList');
          if (notificationsList) {
            const notifications = getNotifications();
            if (notifications.length === 0) {
              notificationsList.innerHTML = '<div class="p-4 text-center text-gray-500">ไม่มีการแจ้งเตือน</div>';
            } else {
              const config = window.elementSdk?.config || defaultConfig;
              const textColor = config.text_color || defaultConfig.text_color;
              
              notificationsList.innerHTML = notifications.map(n => `
                <div class="p-4 hover:bg-gray-50 cursor-pointer" onclick="viewRequestDetail('${n.id}')">
                  <p class="font-medium" style="color: ${textColor};">${n.project_name}</p>
                  <p class="text-sm text-gray-600 mt-1">${n.notes}</p>
                  <p class="text-xs text-gray-400 mt-1">${new Date(n.submitted_date).toLocaleDateString('th-TH')}</p>
                </div>
              `).join('');
            }
          }
        }
      }
    };

    async function onConfigChange(config) {
      const systemTitle = config.system_title || defaultConfig.system_title;
      const orgName = config.organization_name || defaultConfig.organization_name;
      const primaryColor = config.primary_color || defaultConfig.primary_color;
      const secondaryColor = config.secondary_color || defaultConfig.secondary_color;
      const textColor = config.text_color || defaultConfig.text_color;
      
      const titleElements = document.querySelectorAll('h1');
      titleElements.forEach(el => {
        if (el.textContent.includes('ระบบ') || el.textContent.includes(defaultConfig.system_title)) {
          el.textContent = systemTitle;
          el.style.color = primaryColor;
        }
      });
      
      const orgElements = document.querySelectorAll('p');
      orgElements.forEach(el => {
        if (el.textContent === defaultConfig.organization_name || el.classList.contains('text-lg')) {
          el.textContent = orgName;
        }
      });
      
      document.querySelectorAll('[style*="background-color"]').forEach(el => {
        const currentBg = el.style.backgroundColor;
        if (currentBg.includes('30, 64, 175') || currentBg.includes('rgb(30, 64, 175)')) {
          el.style.backgroundColor = primaryColor;
        }
      });
      
      document.querySelectorAll('[style*="color"]').forEach(el => {
        const currentColor = el.style.color;
        if (currentColor.includes('31, 41, 55') || currentColor.includes('rgb(31, 41, 55)')) {
          el.style.color = textColor;
        }
      });
    }

    document.addEventListener('DOMContentLoaded', async () => {
      const initResult = await window.dataSdk.init(dataHandler);
      
      if (!initResult.isOk) {
        console.error('Failed to initialize Data SDK');
        return;
      }
      
      if (window.elementSdk) {
        window.elementSdk.init({
          defaultConfig,
          onConfigChange,
          mapToCapabilities: (config) => ({
            recolorables: [
              {
                get: () => config.primary_color || defaultConfig.primary_color,
                set: (value) => {
                  config.primary_color = value;
                  window.elementSdk.setConfig({ primary_color: value });
                }
              },
              {
                get: () => config.secondary_color || defaultConfig.secondary_color,
                set: (value) => {
                  config.secondary_color = value;
                  window.elementSdk.setConfig({ secondary_color: value });
                }
              },
              {
                get: () => config.text_color || defaultConfig.text_color,
                set: (value) => {
                  config.text_color = value;
                  window.elementSdk.setConfig({ text_color: value });
                }
              },
              {
                get: () => config.success_color || defaultConfig.success_color,
                set: (value) => {
                  config.success_color = value;
                  window.elementSdk.setConfig({ success_color: value });
                }
              },
              {
                get: () => config.warning_color || defaultConfig.warning_color,
                set: (value) => {
                  config.warning_color = value;
                  window.elementSdk.setConfig({ warning_color: value });
                }
              }
            ],
            borderables: [],
            fontEditable: undefined,
            fontSizeable: undefined
          }),
          mapToEditPanelValues: (config) => new Map([
            ["system_title", config.system_title || defaultConfig.system_title],
            ["organization_name", config.organization_name || defaultConfig.organization_name]
          ])
        });
      }
      
      renderApp();
      
      document.addEventListener('submit', (e) => {
        if (e.target.id === 'loginForm') {
          handleLogin(e);
        }
      });
    });

    window.handleLogin = handleLogin;
    window.logout = logout;
    window.togglePassword = togglePassword;
    window.toggleNotifications = toggleNotifications;
    window.showView = showView;
    window.showAddRequestForm = showAddRequestForm;
    window.viewRequestDetail = viewRequestDetail;
    window.approveRequest = approveRequest;
    window.requestDisbursement = requestDisbursement;
    window.updateRequestStatus = updateRequestStatus;
    window.proposeAfterEdit = proposeAfterEdit;
    window.requestAfterEdit = requestAfterEdit;
    window.showReturnDocumentForm = showReturnDocumentForm;
    window.editRequest = editRequest;
    window.confirmDeleteRequest = confirmDeleteRequest;
    window.deleteRequest = deleteRequest;
    window.showAddDepartmentUserForm = showAddDepartmentUserForm;
    window.editDepartmentUser = editDepartmentUser;
    window.confirmDeleteDepartmentUser = confirmDeleteDepartmentUser;
    window.deleteDepartmentUser = deleteDepartmentUser;
  </script>
 <script>(function(){function c(){var b=a.contentDocument||a.contentWindow.document;if(b){var d=b.createElement('script');d.innerHTML="window.__CF$cv$params={r:'9a61293531e2f8cc',t:'MTc2NDQwOTQwOS4wMDAwMDA='};var a=document.createElement('script');a.nonce='';a.src='/cdn-cgi/challenge-platform/scripts/jsd/main.js';document.getElementsByTagName('head')[0].appendChild(a);";b.getElementsByTagName('head')[0].appendChild(d)}}if(document.body){var a=document.createElement('iframe');a.height=1;a.width=1;a.style.position='absolute';a.style.top=0;a.style.left=0;a.style.border='none';a.style.visibility='hidden';document.body.appendChild(a);if('loading'!==document.readyState)c();else if(window.addEventListener)document.addEventListener('DOMContentLoaded',c);else{var e=document.onreadystatechange||function(){};document.onreadystatechange=function(b){e(b);'loading'!==document.readyState&&(document.onreadystatechange=e,c())}}}})();</script></body>
</html>
