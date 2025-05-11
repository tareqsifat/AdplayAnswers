# Part 1 answer:
 - here is the github link of part 1 laravel project
 [Part One Answer](https://github.com/tareqsifat/product_adplay)
# Part 2 answer:

## Answer to the Question No. 1

There are two options to resolve the error

### Option 1: Use MySQL Binlogs (If Enabled)

Binlogs are like a "black box" of your database—they record every change. If enabled, you can "rewind" the accidental update.

Steps:

- Find the Accident  
  Identify the exact time the bad UPDATE ran (e.g., around 9:50 AM on April 30).
- Use a tool like `binlog2sql` (recommended) or `mysqlbinlog` to generate SQL that reverses the accident.
- Example command:

  ```bash
  binlog2sql --flashback --start-datetime='2025-04-30 09:50:00' --stop-datetime='2025-04-30 09:55:00' -d your_database -t users > revert.sql
  ```

  - This creates a file (`revert.sql`) with UPDATE statements that restore the original email_verified values.
  - Run the generated SQL in production. For example:
  ```sql 
    UPDATE users SET email_verified = 0 WHERE user_id = 45;
    ```
    Do the same for all unverified users.
    ### Option 1: Use MySQL Binlogs (If Enabled)
    - Create a Snapshot of the Last Good State:
      ```sql
        CREATE TEMPORARY TABLE last_good_state AS 
        SELECT user_id, old_email_verified  
        FROM user_audit  
        WHERE updated_at < '2025-04-30 09:50:00';
    
    - Restore Data from the Snapshot:

    ```sql
        UPDATE users  
        JOIN last_good_state ON users.user_id = last_good_state.user_id  
        SET users.email_verified = last_good_state.old_email_verified;
## Answer to the Question No. 2

- The task is to Write a Laravel code to compare campaign spend totals across MySQL and MSSQL here is my code.

```php
// Laravel code

public function compareSpend() {  
    // Fetch MySQL data (campaigns)  
    $mysqlSpend = DB::connection('mysql')  
        ->table('campaigns')  
        ->select(  
            'campaigncode',  
            DB::raw('SUM(fixed_price * campaignshow) as total_spend')  
        )  
        ->groupBy('campaigncode')  
        ->get()  
        ->keyBy('campaigncode');  

    // Fetch MSSQL data (billing)  
    $mssqlSpend = DB::connection('mssql')  
        ->table('billing_records')  
        ->select(  
            'campaigncode',  
            DB::raw('SUM(invoiced_amount) as total_invoiced')  
        )  
        ->groupBy('campaigncode')  
        ->get()  
        ->keyBy('campaigncode');  

    // Compare totals  
    $mismatches = [];  
    foreach ($mysqlSpend as $code => $mysql) {  
        $mssql = $mssqlSpend->get($code);  
        if (!$mssql || $mysql->total_spend != $mssql->total_invoiced) {  
            $mismatches[] = [  
                'campaigncode' => $code,  
                'mysql_spend' => $mysql->total_spend,  
                'mssql_invoiced' => $mssql->total_invoiced ?? 0  
            ];  
        }  
    }  

    return $mismatches;  
}  
```
## Answer to the Question No. 3
- here is my sql query to identify invalid campaigns.
```sql
SELECT id, campaign_name, start_date, end_date  
FROM campaigns  
WHERE start_date > end_date;  
```
- database constrain solution
```sql
ALTER TABLE campaigns  
ADD CONSTRAINT chk_dates CHECK (start_date <= end_date);  
```
- application-level validation
```php
## in the campaigns create method
  if (!$start_date || !$end_date) {  
    throw new Error('Both start_date and end_date are required.');  
  }  
  ```
## Answer to the Question No. 4
- Campaigns with NULL or non-matching user_id
```sql
    SELECT c.*  
    FROM campaigns c  
    LEFT JOIN users u ON c.user_id = u.id  
    WHERE u.id IS NULL OR c.user_id IS NULL;  
```
- Campaigns with Missing campaign_category_id:
```sql
    SELECT c.*  
    FROM campaigns c  
    LEFT JOIN campaign_categories cc ON c.campaign_category_id = cc.id  
    WHERE cc.id IS NULL;  
```
## Answer to the Question No. 5
- Daily report
```sql
SELECT  
    id AS campaign_id,  
    campaign_name,  
    DATE(created_at) AS day,  
    (fixed_price * campaignshow) AS total_spend  
FROM campaigns  
WHERE status = 1  
  AND created_at >= CURDATE() - INTERVAL 7 DAY  
GROUP BY day, id  
ORDER BY day DESC;  
```
## Answer to the Question No. 6
- Publisher Dashboard
```sql
SELECT *  
FROM campaigns  
WHERE publishercode = 'PUB123'  
  AND publisher_selection = 2  
  AND created_at >= CURDATE() - INTERVAL 7 DAY  
ORDER BY created_at DESC;  
```

## Answer to the Question No. 7

- Under-Spending Campaigns
``` sql
SELECT  
    id,  
    campaign_name,  
    fixed_price * campaignshow AS total_spend,  
    total_budget  
FROM campaigns  
WHERE CURRENT_DATE BETWEEN start_date AND end_date  
  AND (fixed_price * campaignshow) < 0.5 * total_budget;  
  ```

## Answer to the Question No. 8
- all campaigns where daily_budget exceeds total_budget
```sql
SELECT id, campaign_name, daily_budget, total_budget  
FROM campaigns  
WHERE daily_budget > total_budget;  
```
- how to enforce this at the DB
```sql 
ALTER TABLE campaigns  
ADD CONSTRAINT chk_budget CHECK (daily_budget <= total_budget);  
```
- app level validation
```php 
## in the campaigns create method
  if (!$daily_budget <= !$total_budget) {  
    throw new Error('Daily budget can\'t be greater than total budget');  
  }  
  ```
## Answer to the Question No. 9
- Overlapping Campaigns
```sql
SELECT  
    a.id AS campaign_a,  
    b.id AS campaign_b,  
    a.user_id,  
    a.campaign_type  
FROM campaigns a  
JOIN campaigns b  
    ON a.user_id = b.user_id  
    AND a.campaign_type = b.campaign_type  
    AND a.id != b.id  
    AND (a.start_date <= b.end_date AND a.end_date >= b.start_date);  
```

## Answer to the Question No. 10

- MongoDB/MySQL Sync Issues

- Missing Campaigns in MySQL:
```sql
SELECT campaigncode  
FROM campaigns  
WHERE campaigncode NOT IN (  
    SELECT campaigncode FROM campaigns_mysql  
);  
```
- Outdated MySQL Timestamps:
```sql
SELECT  
    mongo.campaigncode,  
    mongo.updated_at AS mongo_updated,  
    mysql.updated_at AS mysql_updated  
FROM campaigns_mongo mongo  
JOIN campaigns_mysql mysql  
    ON mongo.campaigncode = mysql.campaigncode  
WHERE mongo.updated_at > mysql.updated_at;  
```
<br>
<br>
<br>
<br>

# Part 3 answer:
### Step-by-Step Procedure to Deploy a Laravel Application on DigitalOcean with Nginx
- Create a DigitalOcean Droplet
  - Go to DigitalOcean Dashboard → Create Droplet.
  - Choose Ubuntu (latest LTS version).
  - Select a plan (e.g., Basic, $5/month).
  - Enable SSH keys for secure login.
  - Finalize and create the Droplet.
  - we assume our site name is my-domain.com
- Connect to the Droplet via SSH
```bash
  ssh root@my_droplet_ip
  ```
- Update System Packages
```bash
  apt update && apt upgrade -y
```
- Install Required Software
  - php and its extensions
  ```bash
  apt install php-fpm php-mysql php-mbstring php-xml php-bcmath php-zip php-curl -y
  ```
  - Composer
  ```bash
  curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
  ```
  - Nginx:
  ```bash
    apt install nginx -y
    ```
  - MySQL:
  ```bash
  apt install mysql-server -y
  mysql_secure_installation 
  ```

- Configure MySQL Database
```mysql
    mysql -u root -p
CREATE DATABASE laravel_db;
CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON laravel_db.* TO 'laravel_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
- Deploy Laravel Application
  - Clone
  ```bash
      git clone https://github.com/yourusername/my-test-laravel-app.git /var/www/my-domain
    ```
  - Install dependencies:
  ```bash
    cd /var/www/laravel && composer install --no-dev
    ```
  - Give proper permission
    ```bash
      chown -R www-data:www-data /var/www/my-domain
      chmod -R 775 /var/www/laravel/storage
      chmod -R 775 /var/www/laravel/bootstrap/cache
      ```
- Configure Environment File
```bash
    cp .env.example .env
    nano .env  
    php artisan key:generate
    php artisan config:cache
  ```
- Configure Nginx Server Block
  - Create a new config file:
  ```bash
      nano /etc/nginx/sites-available/laravel
    ```
  - Add the following configuration
  ```bash
      server {
          listen 80;
          server_name my-domain.com www.my-domain.com;
          root /var/www/laravel/public;

          add_header X-Frame-Options "SAMEORIGIN";
          add_header X-Content-Type-Options "nosniff";

          index index.php;

          location / {
              try_files $uri $uri/ /index.php?$query_string;
          }

          location ~ \.php$ {
              include snippets/fastcgi-php.conf;
              fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;  # Adjust PHP version
          }

          location ~ /\.(?!well-known).* {
              deny all;
          }
      }
    ```
  - Enable the site and disable default:
    ```bash
        ln -s /etc/nginx/sites-available/laravel /etc/nginx/sites-enabled/
        rm /etc/nginx/sites-enabled/default
      ```
  - Test and reload Nginx:
    ```bash
        nginx -t
        systemctl reload nginx
      ```
- Secure with Let’s Encrypt SSL
```bash
    apt install certbot python3-certbot-nginx -y
    certbot --nginx -d your_domain.com
  ```
    
- Test the Application
<br><p> Open a browser and hit my-domain.com to see the result</p>
