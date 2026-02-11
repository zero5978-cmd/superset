-- ===============================================
-- 智能儀表板 SQL 系統
-- ===============================================

-- 1. 建立資料庫
CREATE DATABASE IF NOT EXISTS smart_dashboard;
USE smart_dashboard;

-- ===============================================
-- 2. 建立基礎資料表
-- ===============================================

-- 用戶表
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    department VARCHAR(50),
    role VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP
);

-- 銷售數據表
CREATE TABLE sales (
    sale_id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    product_name VARCHAR(100),
    category VARCHAR(50),
    amount DECIMAL(10, 2),
    quantity INT,
    sale_date DATE,
    region VARCHAR(50),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- 網站訪問數據表
CREATE TABLE website_analytics (
    visit_id INT PRIMARY KEY AUTO_INCREMENT,
    page_url VARCHAR(255),
    visitor_ip VARCHAR(45),
    visit_date DATE,
    visit_time TIME,
    session_duration INT, -- 秒數
    device_type VARCHAR(20), -- mobile, desktop, tablet
    browser VARCHAR(50)
);

-- 客戶表
CREATE TABLE customers (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_name VARCHAR(100),
    email VARCHAR(100),
    phone VARCHAR(20),
    country VARCHAR(50),
    registration_date DATE,
    total_spent DECIMAL(10, 2) DEFAULT 0,
    status VARCHAR(20) -- active, inactive, vip
);

-- 產品庫存表
CREATE TABLE inventory (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(100),
    category VARCHAR(50),
    stock_quantity INT,
    reorder_level INT,
    unit_price DECIMAL(10, 2),
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ===============================================
-- 3. 插入測試數據
-- ===============================================

INSERT INTO users (username, email, department, role, last_login) VALUES
('alice_wang', 'alice@company.com', '業務部', 'manager', '2024-02-10 14:30:00'),
('bob_chen', 'bob@company.com', '行銷部', 'staff', '2024-02-11 09:15:00'),
('carol_liu', 'carol@company.com', '業務部', 'staff', '2024-02-09 16:45:00'),
('david_lin', 'david@company.com', 'IT部', 'admin', '2024-02-11 08:00:00');

INSERT INTO sales (user_id, product_name, category, amount, quantity, sale_date, region) VALUES
(1, '筆記型電腦', '電子產品', 35000.00, 2, '2024-02-01', '北區'),
(1, '無線滑鼠', '配件', 890.00, 5, '2024-02-01', '北區'),
(2, '機械鍵盤', '配件', 2500.00, 3, '2024-02-02', '中區'),
(3, '顯示器', '電子產品', 8500.00, 4, '2024-02-03', '南區'),
(1, '筆記型電腦', '電子產品', 35000.00, 1, '2024-02-05', '北區'),
(2, '平板電腦', '電子產品', 15000.00, 2, '2024-02-06', '東區'),
(3, '藍牙耳機', '配件', 1200.00, 10, '2024-02-07', '南區'),
(1, '滑鼠墊', '配件', 250.00, 8, '2024-02-08', '北區');

INSERT INTO customers (customer_name, email, country, registration_date, total_spent, status) VALUES
('張三', 'zhang@email.com', '台灣', '2023-01-15', 45000.00, 'vip'),
('李四', 'li@email.com', '台灣', '2023-06-20', 12000.00, 'active'),
('王五', 'wang@email.com', '香港', '2024-01-10', 3000.00, 'active'),
('趙六', 'zhao@email.com', '新加坡', '2022-11-05', 85000.00, 'vip');

INSERT INTO inventory (product_name, category, stock_quantity, reorder_level, unit_price) VALUES
('筆記型電腦', '電子產品', 25, 10, 35000.00),
('顯示器', '電子產品', 15, 5, 8500.00),
('機械鍵盤', '配件', 50, 20, 2500.00),
('無線滑鼠', '配件', 100, 30, 890.00),
('藍牙耳機', '配件', 8, 15, 1200.00); -- 低於補貨點

-- ===============================================
-- 4. 儀表板查詢 - KPI 指標
-- ===============================================

-- 【總覽面板】今日銷售總額
CREATE VIEW dashboard_today_sales AS
SELECT 
    COUNT(*) as total_orders,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_order_value,
    SUM(quantity) as total_items_sold
FROM sales
WHERE sale_date = CURDATE();

-- 【總覽面板】本月銷售總額
CREATE VIEW dashboard_monthly_sales AS
SELECT 
    YEAR(sale_date) as year,
    MONTH(sale_date) as month,
    COUNT(*) as total_orders,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_order_value
FROM sales
WHERE YEAR(sale_date) = YEAR(CURDATE()) 
  AND MONTH(sale_date) = MONTH(CURDATE())
GROUP BY YEAR(sale_date), MONTH(sale_date);

-- ===============================================
-- 5. 儀表板查詢 - 銷售分析
-- ===============================================

-- 【銷售分析】各類別銷售排行
CREATE VIEW dashboard_sales_by_category AS
SELECT 
    category,
    COUNT(*) as order_count,
    SUM(amount) as total_revenue,
    SUM(quantity) as total_quantity,
    ROUND(SUM(amount) / SUM(SUM(amount)) OVER () * 100, 2) as revenue_percentage
FROM sales
GROUP BY category
ORDER BY total_revenue DESC;

-- 【銷售分析】各地區銷售表現
CREATE VIEW dashboard_sales_by_region AS
SELECT 
    region,
    COUNT(*) as order_count,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_order_value,
    RANK() OVER (ORDER BY SUM(amount) DESC) as revenue_rank
FROM sales
GROUP BY region
ORDER BY total_revenue DESC;

-- 【銷售分析】每日銷售趨勢（最近30天）
CREATE VIEW dashboard_daily_trend AS
SELECT 
    sale_date,
    COUNT(*) as orders,
    SUM(amount) as revenue,
    SUM(quantity) as items_sold
FROM sales
WHERE sale_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY sale_date
ORDER BY sale_date;

-- ===============================================
-- 6. 儀表板查詢 - 業績排行
-- ===============================================

-- 【業績排行】業務員銷售排行榜
CREATE VIEW dashboard_top_sellers AS
SELECT 
    u.username,
    u.department,
    COUNT(s.sale_id) as total_sales,
    SUM(s.amount) as total_revenue,
    AVG(s.amount) as avg_sale_value,
    DENSE_RANK() OVER (ORDER BY SUM(s.amount) DESC) as rank_position
FROM users u
JOIN sales s ON u.user_id = s.user_id
GROUP BY u.user_id, u.username, u.department
ORDER BY total_revenue DESC;

-- 【業績排行】熱銷產品 TOP 10
CREATE VIEW dashboard_top_products AS
SELECT 
    product_name,
    category,
    COUNT(*) as times_sold,
    SUM(quantity) as total_quantity,
    SUM(amount) as total_revenue,
    ROUND(AVG(amount/quantity), 2) as avg_unit_price
FROM sales
GROUP BY product_name, category
ORDER BY total_revenue DESC
LIMIT 10;

-- ===============================================
-- 7. 儀表板查詢 - 客戶分析
-- ===============================================

-- 【客戶分析】客戶價值分析
CREATE VIEW dashboard_customer_value AS
SELECT 
    status,
    COUNT(*) as customer_count,
    SUM(total_spent) as total_revenue,
    AVG(total_spent) as avg_spent_per_customer,
    MAX(total_spent) as highest_spent,
    MIN(total_spent) as lowest_spent
FROM customers
GROUP BY status
ORDER BY total_revenue DESC;

-- 【客戶分析】VIP 客戶列表
CREATE VIEW dashboard_vip_customers AS
SELECT 
    customer_name,
    email,
    country,
    total_spent,
    registration_date,
    DATEDIFF(CURDATE(), registration_date) as days_as_customer
FROM customers
WHERE status = 'vip'
ORDER BY total_spent DESC;

-- ===============================================
-- 8. 儀表板查詢 - 庫存警報
-- ===============================================

-- 【庫存管理】低庫存警報
CREATE VIEW dashboard_low_stock_alert AS
SELECT 
    product_id,
    product_name,
    category,
    stock_quantity,
    reorder_level,
    (reorder_level - stock_quantity) as shortage,
    unit_price,
    CASE 
        WHEN stock_quantity = 0 THEN '缺貨'
        WHEN stock_quantity < reorder_level THEN '庫存不足'
        ELSE '正常'
    END as stock_status
FROM inventory
WHERE stock_quantity <= reorder_level
ORDER BY stock_quantity ASC;

-- 【庫存管理】庫存價值統計
CREATE VIEW dashboard_inventory_value AS
SELECT 
    category,
    COUNT(*) as product_count,
    SUM(stock_quantity) as total_units,
    SUM(stock_quantity * unit_price) as total_value,
    AVG(stock_quantity * unit_price) as avg_value_per_product
FROM inventory
GROUP BY category
ORDER BY total_value DESC;

-- ===============================================
-- 9. 進階查詢 - 組合分析
-- ===============================================

-- 【綜合儀表板】完整 KPI 總覽
SELECT 
    '銷售總覽' as metric_category,
    (SELECT COUNT(*) FROM sales WHERE sale_date = CURDATE()) as today_orders,
    (SELECT COALESCE(SUM(amount), 0) FROM sales WHERE sale_date = CURDATE()) as today_revenue,
    (SELECT COUNT(*) FROM sales WHERE MONTH(sale_date) = MONTH(CURDATE())) as month_orders,
    (SELECT COALESCE(SUM(amount), 0) FROM sales WHERE MONTH(sale_date) = MONTH(CURDATE())) as month_revenue,
    (SELECT COUNT(*) FROM customers WHERE status = 'active') as active_customers,
    (SELECT COUNT(*) FROM inventory WHERE stock_quantity <= reorder_level) as low_stock_items;

-- 【月度對比】本月 vs 上月
SELECT 
    'Current Month' as period,
    COUNT(*) as orders,
    SUM(amount) as revenue,
    AVG(amount) as avg_order
FROM sales
WHERE YEAR(sale_date) = YEAR(CURDATE()) 
  AND MONTH(sale_date) = MONTH(CURDATE())
UNION ALL
SELECT 
    'Last Month' as period,
    COUNT(*) as orders,
    SUM(amount) as revenue,
    AVG(amount) as avg_order
FROM sales
WHERE sale_date >= DATE_FORMAT(DATE_SUB(CURDATE(), INTERVAL 1 MONTH), '%Y-%m-01')
  AND sale_date < DATE_FORMAT(CURDATE(), '%Y-%m-01');

-- ===============================================
-- 10. 查詢所有儀表板視圖
-- ===============================================

-- 使用範例：查看今日銷售
SELECT * FROM dashboard_today_sales;

-- 使用範例：查看本月銷售
SELECT * FROM dashboard_monthly_sales;

-- 使用範例：查看銷售排行
SELECT * FROM dashboard_top_sellers;

-- 使用範例：查看庫存警報
SELECT * FROM dashboard_low_stock_alert;

-- 使用範例：查看熱銷產品
SELECT * FROM dashboard_top_products;

-- ===============================================
-- 11. 權限設定（可選）
-- ===============================================

-- 創建儀表板專用唯讀用戶
-- CREATE USER 'dashboard_user'@'localhost' IDENTIFIED BY 'secure_password';
-- GRANT SELECT ON smart_dashboard.* TO 'dashboard_user'@'localhost';
-- FLUSH PRIVILEGES;

-- ===============================================
-- 完成！
-- ===============================================
