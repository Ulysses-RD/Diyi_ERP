# MSSQL連線池 Pooling 測試

```xml
  <connectionStrings>
    <add name="SQLConnectionString_s" connectionString="Data Source=192.168.3.115;Initial Catalog=erp_dbSQL;User ID=diyiuser;Password=xxxxxxxxx" providerName="System.Data.SqlClient" />
    <add name="SQLConnectionString" connectionString="Server=tcp:xxx.XXX.xxx.x,11433;Initial Catalog=erp_dbSQL;Persist Security Info=False;User ID=xxxxxx;Password=xxxxxxxx;MultipleActiveResultSets=False;Encrypt=true ;TrustServerCertificate=true;Connection Timeout=350;" />
  </connectionStrings>
```

我們的程式碼在 `App.Config`沒有設定 `Pooling=true` ，所以是預設模式 : 

預設的連接池中:

1. **連接池默認最大連接數為 100**。
2. **空閒連接的最大存活時間為 4-8 分鐘**。(自動回收)
3. **連線池是 ADO.NET SqlClient 預設啟用的功能**，即使你沒有在連線字串中明確設定，連線池依然會啟用。

- 如果你想控制連線池行為，可以在連線字串中加上：

  - `Pooling=true` 或 `Pooling=false`（預設是 `true`）
  - `Min Pool Size=數字`
  - `Max Pool Size=數字`
  - `Connection Lifetime=秒數`


## 我這邊測試將App.config 設定 `Max Pool Siz= 50` 點擊查詢和撈取資料(純手動)。

```xml
    <add name="SQLConnectionString" connectionString="Server=tcp:xxx.XXX.xxx.x,11433;Initial Catalog=erp_dbSQL;Persist Security Info=False;User ID=xxxxxx;Password=xxxxxxxx;MultipleActiveResultSets=False;Encrypt=true ;TrustServerCertificate=true;Connection Timeout=350 Timeout=10 ;Pooling=true;Min Pool Size = 5;Max Pool Size=50;" />;"

```

使用前人的 **`InitDB`** 去進行，等於每次開啟連線都會使用到 兩次conn.Open(); 最後只會關閉一次
```C#
public void InitDB()
{
    ConnString = CisMyGlobal.SQLConectionString;
    conn = new SqlConnection(ConnString);
    conn.Open();
}

// 其他funtion 資料表查詢邏輯
conn.Open();

```

出現連線池超過數量 ( 代表連線次數超過上線 )

![image](https://github.com/user-attachments/assets/b2509f9e-5253-4160-a0e8-895d9366762d)

😆當連線池到達設定值 50 時，ERP畫面整個當機不能動作。

![image](https://github.com/user-attachments/assets/1a874389-8f22-4f01-a250-f912ed755acb)

---

# 查詢最大 Mssql 連線數語法 

```SQL
SELECT COUNT(*) FROM sys.dm_exec_connections WHERE session_id <> @@SPID
```

```SQL
SELECT program_name, COUNT(*) AS ConnectionCount
FROM sys.dm_exec_sessions
GROUP BY program_name
ORDER BY ConnectionCount DESC
```


和「最大連線池（Max Pool Size）」沒有直接關係，但有間接關聯。

詳細說明
1. 這個查詢的意義
•	這是查詢目前 SQL Server 上所有連線（Connection）數量，不分來源、不分應用程式。
•	包含所有用戶端（你的程式、同事的程式、SSMS、排程、API...）的連線。
•	這個數字不是單一應用程式的連線池數量。
2. Max Pool Size 是什麼？
•	Max Pool Size 是單一應用程式（單一連線字串）在 .NET 連線池中可同時存在的最大連線數。
•	例如：Max Pool Size=50，代表你的應用程式最多同時有 50 條連線在連線池中。
3. 兩者的關係
•	如果你的應用程式大量開啟連線、又沒正確釋放，會讓 SQL Server 上的連線數增加，但這個查詢查到的是「所有來源」的總和。
•	若你有多個應用程式、或同事也在用，這個查詢的數字會更大。
•	Max Pool Size 只會限制你這個應用程式的連線池，不會限制 SQL Server 的總連線數。
4. 如何觀察 Max Pool Size 是否被用滿？
•	如果你的程式出現「Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool」這類錯誤，代表 Max Pool Size 被用滿。
•	你可以用 .NET 的 PerformanceCounter 監控本機的「NumberOfPooledConnections」和「NumberOfActiveConnections」。
•	或在程式內部統計 SqlConnection 的開啟/關閉狀態。

---
結論
•	這個 SQL 查詢只能看到 SQL Server 目前所有連線的總數，無法直接反映「你的應用程式的連線池是否滿」。
•	Max Pool Size 是應用程式端的設定，和 SQL Server 端的連線總數是兩個不同層次的概念。

