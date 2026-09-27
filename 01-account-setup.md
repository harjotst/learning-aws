# Module 01 · Account Setup: Create, Secure, and Open Your Terminal

**Time: about 20 minutes.**
**You'll need:** an email address you can check right now, a phone, and a credit/debit card
(AWS requires one even for free usage).

**By the end of this module you will have:**
- An AWS account with a spending alert, so you can't get a surprise bill
- A secured "root" login you'll put away, and a separate everyday "admin" login
- A terminal inside AWS (CloudShell) that's already logged in and ready for every lab in this course

---

## Part 1: Concepts (5 min read)

### 1.1 Regions and Availability Zones

AWS has data centers all over the world, grouped like this:

```
AWS
 ├── Region: us-east-1  (Northern Virginia, USA)
 │     ├── Availability Zone us-east-1a   ← one or more data center buildings
 │     ├── Availability Zone us-east-1b   ← separate buildings, separate power and network,
 │     ├── Availability Zone us-east-1c     a few miles apart, connected by fast private links
 │     └── ... (us-east-1 has six)
 ├── Region: eu-west-1  (Ireland)
 │     └── ...
 └── ... about 35 regions worldwide
```

- A **Region** is a geographic area. Almost everything you create lives in **one** region and
  only shows up when that region is selected. If you create a server in `us-east-1` and then look
  in `eu-west-1`, you won't see it. **This is the #1 "where did my stuff go?" confusion for
  beginners.** In this course we use **`us-east-1`** for everything.
- An **Availability Zone (AZ)** is one isolated location inside a region. Fires, floods, and power
  cuts happen, but they almost never hit two AZs at once. So:

> **The most important design rule in AWS: to keep something running when things break, run it
> in at least two Availability Zones.** You'll do exactly this in Module 04.

- A few services are **global** (not tied to a region). The main one is **IAM**, the permission
  system you'll use in Module 02.

### 1.2 Accounts and the root user

- An **AWS account** is your container for everything: every resource you create belongs to one
  account, and one account gets one bill. It's identified by a 12-digit **account ID** like
  `123456789012`.
- When you sign up, you create the **root user**: your email address plus a password. **Root can
  do absolutely anything**, including closing the account and changing payment details. If
  someone steals it, they own your account. So the standard practice is:
  1. Protect root with **MFA** (multi-factor authentication: a code from your phone in addition
     to the password).
  2. Create a separate **admin user** for everyday work.
  3. Almost never log in as root again.

### 1.3 How you pay

AWS bills **per use**: per hour a server runs, per GB stored, per million requests. Nothing costs
money until you create it, and most things stop costing money when you delete them.

**What this course costs:** Every lab tells you exactly what to delete at the end. If you follow
those steps the same day, the whole course costs **well under $1**, and new accounts get free
credits that cover it. The only things in this course that charge by the hour are the two small
servers and the load balancer in Module 04, and you'll delete them after about 30 minutes.

### 1.4 The shared responsibility model

AWS is responsible for the physical security of data centers, the hardware, and the software
that runs its services. **You** are responsible for how you configure things: who has access, what's open to
the internet, and what data you store. Almost every AWS security incident you'll read about is a
customer configuration mistake, like a storage bucket left public or a password leaked in code,
not AWS being hacked.

---

## Part 2: Create the account (8 min)

> The AWS sign-up pages change their layout occasionally. If a screen looks slightly different
> from what's described, look for the same fields; the information asked for is the same.

### Step 1: Start the sign-up

1. Open **https://aws.amazon.com** in your browser.
2. Click **Create an AWS Account** (top-right; it may say **Sign up** or **Create account**).

### Step 2: Email, account name, verification

1. **Root user email address**: an email you control and will keep long-term.
2. **AWS account name**: anything, e.g. `harjot-learning`. You can change it later.
3. Click **Verify email address**. AWS emails you a code. Enter it and click **Verify**.

### Step 3: Root password

Create a strong password. Save it in a password manager. This is the **root** password.

### Step 4: Choose a plan

AWS may ask you to pick **Free plan** or **Paid plan**.
- **Free plan**: you get free credits (currently up to $200) and **you cannot be charged**
  beyond them; the account is limited to about 6 months unless you upgrade. **Pick this for
  learning.**
- If a lab later says a feature isn't available on the free plan, you can upgrade then, and your
  credits still apply.

(If you don't see this choice, your account is on the standard model: you're billed for usage,
and the budget alert in Part 4 is your safety net.)

### Step 5: Contact information

- Choose **Personal** (unless this is for a company).
- Fill in name, phone, and address. Tick the customer agreement box. Click **Continue**.

### Step 6: Billing information

Enter a card. AWS may place a temporary charge of about $1 to verify it; it's refunded.

### Step 7: Identity verification

Choose text message (SMS) or voice call, enter the code AWS sends you.

### Step 8: Support plan

Choose **Basic support – Free**. Click **Complete sign up**.

**What you should see:** a page saying your account is being activated. Activation usually takes
a few minutes (occasionally up to a day). You'll get an email when it's ready.

---

## Part 3: Secure the root user (4 min)

### Step 1: Sign in as root

1. Go to **https://console.aws.amazon.com**.
2. Choose **Root user**, enter your email, click **Next**, enter the password.

> AWS now **requires** MFA for root users. If it asks you to register an MFA device right after
> sign-in, that's this step. Follow its prompts, which match the steps below.

### Step 2: Turn on MFA

1. Install an authenticator app on your phone if you don't have one: **Google Authenticator**,
   **Microsoft Authenticator**, **Authy**, or **1Password** all work.
2. In the AWS console, click your **account name** in the top-right corner → **Security credentials**.
3. In the **Multi-factor authentication (MFA)** section, click **Assign MFA device**.
4. **Device name**: `root-phone`. Select **Authenticator app**. Click **Next**.
5. Click **Show QR code**. In your phone app, tap **+** / **Add account** / **Scan QR code**, and
   scan it.
6. Your app now shows a 6-digit code that changes every 30 seconds. Type the current code into
   **MFA code 1**, **wait for it to change**, then type the new code into **MFA code 2**.
7. Click **Add MFA**.

**What you should see:** your device listed under Multi-factor authentication. From now on,
signing in as root asks for the phone code.

### Step 3: Let your admin user see billing

By default, only root can view bills and budgets. Change that so you rarely need root:

1. Click your **account name** (top-right) → **Account**.
2. Scroll to **IAM user and role access to Billing information** → click **Edit**.
3. Tick **Activate IAM Access** → **Update**.

---

## Part 4: Set a budget alert (3 min)

This is your protection against surprise charges. Do not skip it.

1. Click the **search bar** at the top of the console, type `Budgets`, and click **Budgets**.
2. Click **Create budget**.
3. Choose **Use a template (simplified)**.
4. Choose the template **Monthly cost budget**.
5. **Budget name**: `monthly-5-dollars`.
6. **Enter your budgeted amount**: `5`.
7. **Email recipients**: your email.
8. Click **Create budget**.

**What you should see:** your budget listed. AWS will email you when your spending reaches 85% and
100% of $5, and when it *forecasts* you'll exceed $5.

> A budget **alerts** you. It does not stop anything. If you ever get that email, open the
> **Billing and Cost Management** page to see which service is costing money, and delete it.

---

## Part 5: Create your everyday admin user (5 min)

This uses **IAM**, the permission system you'll study properly in Module 02. For now, just follow
the steps.

### Step 1: Open IAM

1. Search bar → type `IAM` → click **IAM**.
2. Notice the region selector in the top-right says **Global**. IAM isn't tied to a region.

### Step 2: Create the user

1. Left menu → **Users** → **Create user**.
2. **User name**: `admin`.
3. Tick **Provide user access to the AWS Management Console**.
4. If you see *"Are you providing console access to a person?"*, choose **I want to create an IAM
   user**. (AWS suggests "Identity Center" instead. That's the better choice for companies with
   many people, but it's more setup. An IAM user is fine for a personal learning account.)
5. **Console password**: choose **Custom password** and type a strong password (different from root).
6. **Untick** "Users must create a new password at next sign-in".
7. Click **Next**.

### Step 3: Give it administrator permissions

1. Choose **Attach policies directly**.
2. In the search box under **Permissions policies**, type `AdministratorAccess`.
3. Tick the box next to **AdministratorAccess** (the one whose description says it provides full access).
4. Click **Next**, then **Create user**.

### Step 4: Save the sign-in URL

The next page shows **Console sign-in URL**, which looks like:
```
https://123456789012.signin.aws.amazon.com/console
```
**Copy it and bookmark it.** IAM users sign in at this URL, not the normal root sign-in page.
The number in it is your **account ID**.

### Step 5: Switch to the admin user

1. Top-right account name → **Sign out**.
2. Open the sign-in URL you just saved.
3. **IAM user name**: `admin`. Password: the one you just set. Sign in.
4. (Recommended) Add MFA to this user too: top-right → **Security credentials** → **Assign MFA
   device** → same steps as for root, device name `admin-phone`.

**From now on, always sign in as `admin`.** Root is only for rare account-level tasks.

---

## Part 6: Choose your region (1 min)

1. In the top-right of the console, next to your account name, there's a region name (for example
   **N. Virginia** or **Ohio**).
2. Click it and select **US East (N. Virginia) us-east-1**.

**Every module assumes `us-east-1`.** If something you created seems to have disappeared, check
this dropdown first.

---

## Part 7: Open CloudShell, your terminal for the whole course (5 min)

**AWS CloudShell** is a Linux terminal that runs in your browser, inside the AWS console. It
comes with the AWS CLI already installed and **already logged in as you**. That means no software to
install, no passwords or keys to configure, and it works the same on Windows, Mac, and Linux.

### Step 1: Launch it

1. In the top bar of the console, click the **CloudShell icon**. It looks like a small terminal
   window `>_`, to the left of the bell icon. (Or search `CloudShell`.)
2. If a welcome box appears, close it.
3. Wait 20-60 seconds while it says it's creating the environment.

**What you should see:** a black panel with a prompt ending in `$`, for example:
```
~ $
```
That's the shell waiting for a command (see Module 00, Part 4).

### Step 2: Ask AWS who you are

Type this and press Enter:
```bash
aws sts get-caller-identity
```

**What you should see:**
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/admin"
}
```

**What just happened:** you called the STS service's `GetCallerIdentity` API, the "who am I?"
of AWS. `Account` is your account ID. `Arn` shows you're the IAM user `admin`. **Whenever a
permission error confuses you, run this first** to check who AWS thinks you are.

**If it fails:** if `Arn` ends in `:root`, you're signed in as root. Sign out and sign in as
`admin` with your sign-in URL.

### Step 3: Set up the course helpers

CloudShell forgets your variables when a session ends (after about 20-30 minutes of
inactivity, or if you close the tab). Your **files** in your home folder are kept. The labs
create lots of IDs you'll need later, so we'll store them in a file that reloads automatically.

Copy and paste this **entire block**, including the last `EOF` line, and press Enter:

```bash
mkdir -p ~/labs
cat >> ~/.bashrc <<'EOF'

# ---- AWS course helpers ----
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
touch ~/lab.env
source ~/lab.env
save() { echo "export $1=\"${!1}\"" >> ~/lab.env; echo "saved: $1=${!1}"; }
cd ~/labs
EOF
source ~/.bashrc
```

**What just happened, line by line:**
- `mkdir -p ~/labs` creates a folder called `labs` in your home folder (`~` means "my home folder").
  All lab files will go here.
- `cat >> ~/.bashrc <<'EOF'` ... `EOF` appends the lines in between to `~/.bashrc`, a file that
  bash runs automatically **every time a new terminal session starts**.
- `export AWS_REGION=us-east-1` tells the AWS CLI to use the `us-east-1` region.
- `touch ~/lab.env` creates an empty file `lab.env` if it doesn't exist yet. This is where saved values go.
- `source ~/lab.env` loads every variable saved in that file.
- `save() { ... }` defines a small helper command called `save`. `save VPC_ID` appends the line
  `export VPC_ID="vpc-0abc..."` to `~/lab.env`. You don't need to understand the syntax inside it.
- `cd ~/labs` starts every session in the labs folder.
- The final `source ~/.bashrc` runs the file now, so you don't have to restart.

### Step 4: Save your account ID

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
save ACCOUNT_ID
```

**What you should see:**
```
saved: ACCOUNT_ID=123456789012
```

**What just happened:** `$( ... )` ran the "who am I" command and kept only the `Account` field as
plain text (see Module 00, Part 5.2). That went into the variable `ACCOUNT_ID`, and `save` wrote it
to `~/lab.env`. Many lab commands use `$ACCOUNT_ID`, because some AWS names must be globally unique
and your account ID is an easy way to make them unique.

### Step 5: Prove that saved values survive a restart

1. In the CloudShell panel, click **Actions** → **New tab** (or the `+` next to the tab name).
2. In the new tab, run:
   ```bash
   echo "Account: $ACCOUNT_ID  Region: $AWS_REGION  Folder: $(pwd)"
   ```
3. **What you should see:** `Account: 123456789012  Region: us-east-1  Folder: /home/cloudshell-user/labs`.

The new session loaded everything automatically. **If CloudShell ever times out during a lab, just
reopen it and continue. Your saved variables come back.** You can close the extra tab.

### CloudShell tips

- **Paste**: `Ctrl+V` (Windows/Linux) or `Cmd+V` (Mac). The first time, the browser may ask for
  permission to paste; allow it. CloudShell may warn you when pasting multiple lines; click **Paste**.
- **Make it bigger**: drag the top edge of the panel up, or click the **maximize** icon.
- **See what you've saved**: `cat ~/lab.env`.
- **Upload/download files**: **Actions** → **Upload file** / **Download file**.
- **Files outside your home folder** are deleted when the session ends. Keep everything in `~/labs`.

---

## Part 8: A quick tour of the console (2 min)

Click around to get your bearings. Don't create anything yet.

1. **Console Home**: click the AWS logo top-left. It shows recently visited services and cost widgets.
2. **Search bar**: the fastest way to reach any service. Type `EC2`, `S3`, `Lambda`.
3. **Region selector** (top-right): check it's still `N. Virginia`.
4. **Account menu** (top-right): Security credentials, Account, Billing, Sign out.
5. Open **EC2** (search `EC2`). The **Resources** box shows counts: 0 instances, 1 security group
   (a default one), and so on. AWS creates a few default items in every new account, which you'll
   meet in Module 03. Now switch the region to **Ohio** and back to **N. Virginia**. Notice that
   the page reloads with that region's resources. That's what "resources are regional" looks like.

---

## Checkpoint

1. You created something in `us-east-1`, but the console shows nothing. What do you check first?
2. Why create an `admin` user instead of using root?
3. Does a budget stop AWS from charging you?
4. CloudShell timed out and you reopened it. Is `$ACCOUNT_ID` still set? Why?
5. What command tells you which identity AWS thinks you are?

<details><summary>Answers</summary>

1. The region selector in the top-right. It's probably set to a different region.
2. Root can do everything, including closing the account, and can't be restricted. Day-to-day work should use an identity whose power can be limited and whose credentials matter less if leaked.
3. No, a budget only sends alerts. (On the Free plan, you can't be charged beyond your credits.)
4. Yes, because `save` wrote it into `~/lab.env`, and `~/.bashrc` reloads that file in every new session.
5. `aws sts get-caller-identity`.
</details>

**Where you are now:** you have a secured account, a spending alert, and a ready terminal. Next,
before creating anything that runs, you need to understand the system that decides what's allowed.

**Next: [Module 02 · IAM](02-iam.md)**
