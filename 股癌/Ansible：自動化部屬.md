# Ansible：核心原則、最佳實踐與企業級應用

## 執行摘要

本文件綜合分析了 Ansible 的核心原則、技術實踐以及企業級應用。Ansible 是一款開源的 IT 自動化工具，專注於設定管理、應用程式部署及任務流程編排。其設計哲學圍繞著三大核心：**簡化性**（Complexity Kills Productivity）、**可讀性**（Optimize for Readability）及**宣告式思維**（Think Declaratively），旨在將複雜的自動化任務轉化為易於理解和維護的 YAML Playbook。

在技術實踐上，本文件詳細闡述了從專案結構、Inventory 管理、變數使用到 Playbook 撰寫的最佳實踐。關鍵建議包括將 Ansible 內容視為程式碼進行版本控制、廣泛使用分組和有意義的名稱來管理主機、分離邏輯與變數以增強靈活性，並謹慎使用指令模組，優先選擇專用模組或開發自訂模組。

在生態系統中，Ansible 並非與 Terraform 等基礎設施即程式碼（IaC）工具競爭，而是作為其互補工具，專注於基礎設施佈建完成後的設定管理（CaC）。典型的 DevOps 工作流程是使用 Terraform 建立雲端資源，再由 Ansible 進行系統設定與應用部署。

對於企業級應用，Red Hat Ansible 自動化平台（AAP）提供了擴展 Ansible 所需的關鍵功能，包括集中式的自動化控制器（Automation Controller）、基於角色的存取控制（RBAC）、憑證管理、強大的工作流程以及詳細的日誌審計與分析報告。此外，本文件亦涵蓋了 Ansible 自動化平台的安全強化指南，強調了從架構規劃、身分驗證、憑證管理到符合 STIG 等安全標準的重要性。成功的自動化不僅是技術的導入，更是一場涉及文化、流程與協作的組織轉型。

\--------------------------------------------------------------------------------

## 1. Ansible 核心哲學 (The Ansible Way)

Ansible 的設計理念和最佳實踐根植於其核心哲學，旨在提升生產力並降低自動化門檻。這些原則不僅是行銷口號，而是貫穿於工具設計與社群實踐中的指導方針。

### 1.1 複雜性扼殺生產力 (Complexity Kills Productivity)

Ansible 堅信過度的複雜性是生產力的敵人。因此，無論是工具本身的設計，還是使用者撰寫的自動化內容，都應力求簡化。這意味著偏好使用多個簡單、專注的 Playbook，而不是一個充滿條件判斷的龐大 Playbook，遵循 Linux「做好一件事」的原則。

### 1.2 為可讀性而優化 (Optimize for Readability)

Ansible 的 Playbook 被設計為人類可讀的 YAML 格式。如果撰寫得當，Playbook 本身就能成為工作流程自動化的最佳文件。這要求開發者注重命名、註解、格式和結構的一致性，使得非技術人員也能大致理解自動化流程的意圖。

### 1.3 宣告式思維 (Think Declaratively)

Ansible 本質上是一個「期望狀態引擎」（Desired State Engine）。使用者應專注於**宣告**系統的最終狀態，而非編寫達到該狀態的逐步指令。Playbook 描述的是「什麼」（What），而 Ansible 負責處理「如何」（How）。試圖在 YAML Playbook 中「寫程式」會違背此原則，並導致失敗。

## 2. Ansible 核心組件與概念

要有效使用 Ansible，必須理解其基本構成元素和運作方式。

| 組件/概念                   | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| **控制主機 (Control Host)** | 安裝並執行 Ansible 的伺服器，從這裡發送指令到受管主機。      |
| **受管主機 (Managed Host)** | 由 Ansible 控制主機管理的遠端伺服器或網路設備。              |
| **Inventory 清單**          | 定義受管主機列表的檔案（預設為 `/etc/ansible/hosts`），支援 INI 或 YAML 格式。主機可以透過 IP 或 FQDN 指定，並可組織成群組。 |
| **模組 (Modules)**          | 可在終端或 Playbook 任務中使用的獨立程式碼單元，用於執行特定操作（如安裝軟體、複製檔案）。模組具有**冪等性 (Idempotent)**，即多次執行同一任務不會改變系統狀態。 |
| **任務 (Tasks)**            | 將一個模組應用於受管主機的動作。單個任務可透過 Ad-hoc 指令執行，或在 Playbook 中定義。 |
| **Playbook**                | 由一個或多個「Play」組成的 YAML 檔案，定義了在哪些主機上執行哪些任務。任務按順序執行。 |
| **處理常式 (Handlers)**     | 特殊的任務，只有在被其他任務透過 `notify` 指令觸發時才會執行，常用於重啟服務等操作。 |
| **角色 (Roles)**            | 一種將複雜 Playbook 分解為多個可重用組件的機制，提供標準化的目錄結構來組織任務、變數、模板等內容。 |
| **集合 (Collections)**      | Ansible 內容的發佈格式，可將角色、模組、外掛程式和 Playbook 打包在一起，便於分享和管理。 |
| **Ad-hoc 指令**             | 在單行指令中執行單個任務的簡單方式，適用於快速操作，無需建立完整的 Playbook。 |
| **Facts**                   | Ansible 透過 `gather_facts` 操作從受管主機收集的系統資訊（如作業系統、IP 地址），這些資訊可以作為變數在 Playbook 中使用。 |

## 3. 工作流程與專案結構最佳實踐

結構化和標準化的工作流程是成功擴展 Ansible 自動化的基礎。

### 3.1 將 Ansible 內容視為程式碼

應像對待應用程式碼一樣對待 Ansible 自動化內容：

- **版本控制**：使用 Git 等工具對所有 Playbook、角色和變數進行版本控制。
- **從簡開始，逐步迭代**：從一個簡單的 Playbook 和靜態 Inventory 開始，隨著需求增長再進行重構和模組化。

### 3.2 建立與執行風格指南

一致性對於可維護性至關重要。團隊應建立風格指南，並強制執行，涵蓋以下方面：

- **命名**：為任務、Play、變數和角色制定清晰、一致的命名規則。
- **格式**：統一的空白、縮排和標籤（Tagging）使用方式。
- **目錄佈局**：遵循標準化的專案結構。

### 3.3 專案佈局模式

根據專案的複雜度和重用性需求，可以採用不同的目錄結構。

#### 基本專案

適用於小型、單一用途的專案。

```
basic-project/
├── inventory/
│   ├── group_vars/
│   │   └── web.yml
│   ├── host_vars/
│   │   └── db1.yml
│   └── hosts
└── site.yml
```

#### 組織型角色

當專案變得複雜時，使用 `roles` 子目錄來組織和封裝相關任務。

```
myapp/
├── roles/
│   ├── myapp/
│   ├── nginx/
│   └── proxy/
└── site.yml
```

#### 共享角色

當角色需要在多個專案之間共享時，使用 `requirements.yml` 檔案來管理外部或私有的角色依賴，並透過 `ansible-galaxy` 進行安裝。

```
myapp/
├── config.yml
├── provision.yml
├── roles/
│   └── requirements.yml
└── site.yml
```

在企業環境中，常見的模式是每個專案對應一個 Git 倉庫，共用的 Inventory 和角色則分別存放在獨立的倉庫中，並透過私有的 Ansible Galaxy 或 Automation Hub 進行管理。

## 4. 編寫高效 Ansible 自動化的關鍵實踐

遵循以下實踐可以顯著提升 Ansible Playbook 的品質、可讀性和可維護性。

### 4.1 Inventory 管理

- **使用有意義的名稱**：為 Inventory 中的節點賦予人類可讀的名稱，而不是直接使用 IP 地址。
- **不佳範例 (Exhibit A)**：`10.1.2.75`
- **建議範例 (Exhibit B)**：`db1 ansible_host=10.1.2.75`
- **廣泛分組**：盡可能多地建立群組，例如按照功能（`[db]`、`[web]`）、環境（`[dev]`、`[prod]`）、地理位置（`[east]`、`[west]`）等維度。這樣可以簡化 Playbook 的目標選擇，並大幅減少條件式任務的使用。
- **動態 Inventory**：在雲端或虛擬化環境中，使用動態 Inventory 外掛程式（如 `amazon.aws.aws_ec2`）作為單一事實來源。這能自動同步基礎設施的變化，減少人為錯誤。

### 4.2 變數管理

- **命名約定**：使用描述性強、獨特且有意義的變數名稱。建議為角色變數加上角色名稱前綴，以避免命名衝突（例如 `apache_max_keepalive`）。
- **分離邏輯與變數**：將任務邏輯與變數定義分開。這不僅使 Playbook 更具可讀性，也增加了靈活性和可重用性。

#### 範例：分離邏輯與變數

**不佳範例 (Exhibit A)**：變數值直接嵌入任務中，且路徑重複。

```yaml
- name: Clone repo
  git:
    dest: /home/{{ username }}/exampleapp
    key_file: /home/{{ username }}/.ssh/id_rsa
    repo: git@github.com:example/apprepo.git
```

**建議範例 (Exhibit B)**：使用 `vars` 區塊定義變數，任務只引用變數。

```yaml
- hosts: nodes
  vars:
    user_home_dir: /home/{{ username }}
    app_dir: "{{ user_home_dir }}/exampleapp"
    deploy_key: "{{ user_home_dir }}/.ssh/id_rsa"
  tasks:
    - name: Clone repo
      git:
        dest: "{{ app_dir }}"
        key_file: "{{ deploy_key }}"
        repo: git@github.com:example/exampleapp.git
```

這樣做的好處是變數定義集中，易於覆蓋，且變數名稱本身就起到了「文件」的作用。

### 4.3 Playbook 與任務撰寫

- **採用原生 YAML 語法**：使用多行（字典）格式來定義模組參數，而不是單行 `key=value` 格式。這使得程式碼更易於垂直閱讀，並能更好地支援複雜參數和編輯器語法高亮。
- **正確範例**： `yaml     - name: install telegraf yum: name: telegraf-{{ telegraf_version }} state: present update_cache: yes     `
- **為所有內容命名**：為 Play、任務和區塊提供簡短、有意義的名稱。這在執行 Playbook 時會提供清晰的輸出，極大地改善使用者回饋和除錯體驗。
- **謹慎使用** `**command**` **與** `**shell**` **模組**：僅在沒有對應的專用 Ansible 模組時才作為最後手段使用。專用模組（如 `user`, `yum`, `service`）是宣告式的、冪等的，並提供更好的錯誤處理和狀態回報。
- **開發自訂模組**：如果頻繁使用 `command` 或 `shell` 模組來與某個系統或 API 互動，應考慮開發一個自訂模組。這能將複雜的指令轉換為簡單、可讀的宣告式任務。
- **使用 Smoke Tests 驗證服務**：僅僅啟動一個服務是不夠的。應使用 `uri` 或其他模組進行「冒煙測試」，以確保服務不僅已啟動，而且能正常回應。
- **分離佈建與設定**：將基礎設施佈建（例如建立虛擬機）與系統設定（例如安裝應用程式）的任務分離到不同的 Playbook 中（如 `provision.yml` 和 `configure.yml`），並透過一個主 Playbook (`site.yml`) 進行協調。

### 4.4 模板 (Templates - Jinja2)

- **保持簡單**：Jinja2 模板功能強大，但應保持其簡單性，主要用於變數替換、簡單的條件判斷和迴圈。避免在模板中實現複雜的邏輯，這些邏輯應在 Ansible Playbook 中處理。
- **標記由 Ansible 管理的檔案**：在模板生成的設定檔頂部加上註解，標明該檔案由 Ansible 自動管理，以防止手動修改。

### 4.5 進階語法

- **條件式 (**`**when**`**)**：用於根據特定條件執行任務。例如，僅在 Apache 未安裝時才安裝 Nginx。
- **迴圈 (**`**with_items**`**)**：用於對一個列表進行迭代，以執行重複性任務，如安裝多個套件。
- **區塊 (**`**block**`**)**：允許將多個任務邏輯上分組，並對整個區塊應用單一的條件判斷或錯誤處理。使用 `rescue` 和 `always` 子句可以實現類似程式語言中的 `try...catch...finally` 結構。
- **標籤 (**`**tags**`**)**：為任務分配標籤，以便在執行 Playbook 時可以選擇性地只執行或跳過帶有特定標籤的任務，提高執行的靈活性。

## 5. Ansible 與其他工具的整合

Ansible 在現代 DevOps 工具鏈中扮演著關鍵角色，並與其他工具協同工作。

### 5.1 Ansible vs. Terraform

Ansible 和 Terraform 並非直接競爭，而是互補的關係。

- **Terraform (基礎設施即程式碼 - IaC)**：專注於**佈建**基礎設施資源（如虛擬機、網路、儲存）。它是宣告式的，透過狀態檔案（stateful）來追蹤和管理資源的生命週期。
- **Ansible (設定即程式碼 - CaC)**：專注於**設定**已佈建的基礎設施（如安裝軟體、管理服務、配置使用者）。它是程序式的（imperative），通常是無狀態的（stateless），透過 SSH 或 WinRM 在目標主機上執行任務。

**典型工作流程**：

1. 使用 **Terraform** 建立雲端基礎設施（例如，AWS EC2 執行個體）。
2. Terraform 將輸出執行個體的 IP 地址等資訊。
3. 將這些資訊動態地提供給 **Ansible** 作為 Inventory。
4. 使用 **Ansible** 在這些新建立的執行個體上安裝和設定應用程式。

### 5.2 CI/CD 整合

Ansible 是實現持續整合與持續交付（CI/CD）流程中的重要一環。

- **與 Jenkins/GitLab CI 的整合**：CI/CD 工具（如 Jenkins）可以在建置流程中觸發 Ansible Playbook，以實現自動化部署到開發、測試或生產環境。例如，當程式碼被合併到主分支時，Jenkins 可以呼叫 Ansible Tower/Controller 的 API 來啟動一個部署作業。
- **使用 Molecule 進行測試**：Molecule 是一個專為測試 Ansible 內容（特別是角色）而設計的框架。它支援在多種環境（如 Docker 容器、虛擬機）中進行測試，確保 Ansible 內容的品質、冪等性和跨平台相容性。在 CI 流程中整合 Molecule 測試，可以在程式碼合併前驗證其正確性。

## 6. 企業級 Ansible：Red Hat Ansible 自動化平台

當自動化規模從個人或小團隊擴展到整個企業時，僅靠命令列工具會面臨協調、安全和可視性等方面的挑戰。Red Hat Ansible 自動化平台（AAP）為此提供了企業級解決方案。

### 6.1 自動化控制器 (Automation Controller)

Automation Controller（前身為 Ansible Tower）是 AAP 的核心，提供了一個集中的 WebUI 和 REST API 來管理和擴展 Ansible 自動化。

- **基於角色的存取控制 (RBAC)**：精細地控制使用者和團隊對 Inventory、專案和憑證的存取權限。
- **憑證管理**：集中且安全地儲存 SSH 金鑰、API Token 等敏感憑證，使用者可以在不直接接觸憑證內容的情況下使用它們。
- **強大的工作流程**：允許將多個 Job Template 鏈接在一起，根據前一個作業的成功或失敗來觸發後續步驟，從而實現複雜的多階段自動化。
- **集中式日誌與審計**：所有自動化作業的執行情況都被詳細記錄，便於追蹤、審計和除錯。
- **推送按鈕式自動化**：提供簡單的介面，讓非技術人員也能安全地執行預先定義好的自動化任務。

### 6.2 平台核心組件

| 組件                             | 描述                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| **Projects**                     | 指向 Playbook 的邏輯集合，通常與一個 Git 倉庫同步，實現自動化內容的版本控制。 |
| **Inventories**                  | 定義自動化執行的目標端點，支援從雲端供應商、CMDB 等來源動態同步。 |
| **Credentials**                  | 安全地管理自動化所需的各種憑證，支援與外部金鑰管理系統整合。 |
| **Job Templates**                | 將 Playbook、Inventory 和 Credentials 結合在一起的可執行單元，標準化了自動化任務的執行方式。 |
| **Workflows**                    | 將多個 Job Template 串聯起來，解決複雜的多步驟自動化問題。   |
| **Execution Environments (EEs)** | 作為容器映像執行的、可移植的 Ansible 執行環境，確保了開發和生產環境的一致性。 |
| **Automation Hub**               | 用於管理、分享和探索經過認證或自訂的 Ansible 內容集合（Collections）的中央儲存庫。 |
| **Analytics & Reporting**        | 提供儀表板和報告，如「自動化計算器」，用於追蹤自動化的投資回報率（ROI）和節省的時間成本。 |

## 7. Ansible 自動化平台安全強化指南

在企業環境中部署 Ansible 自動化平台時，必須採取嚴格的安全措施。

### 7.1 規劃與架構考量

- **網路與防火牆**：遵循最小權限原則，嚴格限制對平台各組件（Controller、Hub、Database）所需埠口的存取。例如，僅允許平台內部伺服器存取資料庫埠口（5432）。
- **身分驗證**：盡可能使用外部身分驗證來源（如 LDAP、SAML、Entra ID），並為管理員帳戶保留本地「緊急」存取權限。
- **憑證管理**：除了平台內建的加密儲存，強烈建議與外部憑證保險庫（如 CyberArk, HashiCorp Vault）整合，以實現更高級別的憑證保護和生命週期管理。

### 7.2 安裝與初始設定

- **專用安裝主機**：使用一個專門的、經過安全強化的 RHEL 主機來執行安裝、升級和備份還原操作。
- **使用 PKI 憑證**：使用由組織內部 CA 簽發的憑證替換預設的自簽署憑證。
- **保護敏感變數**：將安裝清單檔案中的敏感密碼（如 `admin_password`）儲存在加密的 Ansible Vault 中。
- **集中式日誌記錄**：設定 Controller 將所有日誌轉發到外部日誌聚合服務（如 Splunk, Elastic Stack），以便進行集中監控和分析。

### 7.3 STIG 合規性考量

對於需要遵循 DISA STIG 安全標準的組織，部署 Ansible 自動化平台時需注意以下幾點：

- **fapolicyd**：STIG 要求啟用 `fapolicyd`，但這與 AAP 的安裝和執行存在衝突。建議將 `fapolicyd` 設定為**寬容模式 (permissive)**，並與安全稽核人員溝通豁免此項控制。
- `**noexec**` **掛載**：STIG 要求 `/tmp`、`/var` 等目錄以 `noexec` 選項掛載。這會導致安裝和執行失敗。安裝時需暫時移除此選項，安裝後可為 `/tmp` 重新啟用，但需將 Controller 的「作業執行路徑」更改到一個沒有 `noexec` 限制的目錄。包含 `/var/lib/awx` 的檔案系統不能以 `noexec` 掛載。
- **基於角色的存取控制 (RBAC)**：充分利用 Controller 內建的 RBAC 功能，實現職責分離，將系統管理員權限限制在最小範圍內。

## 8. 企業案例研究

多家大型企業已成功利用 Ansible 自動化平台推動其 DevOps 和 IT 自動化轉型。

- **法國興業銀行 (Societe Generale)**：利用 Ansible 進行應用程式堆疊（NodeJS + ElasticSearch）的佈建。透過 Ansible Tower（現為 Controller），實現了開發（Dev）與維運（Ops）團隊的職責分離。開發人員可以透過 Jenkins 觸發部署，但最終控制權仍在維運團隊手中，同時 Tower 提供了完整的可追溯性。
- **荷蘭國際集團 (ING)**：在超過 10,000 個節點的規模上部署 Ansible Tower，用於 DevOps 和 CI/CD 流程。成功將平台遷移到現代化的 Windows 和 Linux 伺服器，促進了內部協作，顯著減少了新功能的上市時間以及生產環境中的錯誤和中斷事件。
- **Amelco**：面臨應用程式不穩定和迴歸問題的挑戰，該公司利用 Ansible Controller 的 REST API，將其整合到 CI/CD 流程中。這使得開發人員能夠進行可靠的測試和整合，確保了所有環境部署的一致性和可重複性，從而加速了交付週期。