# Module 00 · Foundations: What You Need to Know Before AWS

**Time: about 20 minutes. No AWS account needed yet.**

This module covers the background that AWS assumes you already have. Every later module uses
these words and symbols, so read it even if some of it feels familiar. When a later module uses
a term, it will say "(see Module 00)" and you can come back here.

---

## Part 1: What "the cloud" actually is

### 1.1 A server

A **server** is just a computer whose job is to answer requests from other computers. When you
open a website, your laptop (the **client**) sends a request over the internet to a server,
and the server sends back the web page (the **response**).

```
Your laptop (client)  ── request: "give me the homepage" ──▶  Server
                      ◀── response: the HTML of the page ───
```

A server is ordinary hardware: CPU, memory (RAM), a disk, and a network connection. It usually
has no screen or keyboard and sits in a rack in a building called a **data center**.

### 1.2 A virtual machine (VM)

One big physical server can be split, by software called a **hypervisor**, into many smaller
pretend computers called **virtual machines**. Each VM has its own operating system (Linux or
Windows), its own share of CPU and memory, and thinks it's a real computer. You can create or
destroy one in seconds without touching any hardware.

### 1.3 The cloud

**"The cloud" means renting computing resources (VMs, storage, databases, networks) from a
company that owns the data centers, and paying only for what you use.** You never see the
hardware. You ask for things through a website or a command, and seconds later they exist.

**AWS (Amazon Web Services)** is the largest such company. Others are Microsoft Azure and Google
Cloud. AWS offers over 200 **services**, each a product that does one kind of job, for example:

| Service name | What it is, in plain words |
|---|---|
| EC2 | Rent virtual machines |
| S3 | Store files |
| RDS | Rent a managed database |
| Lambda | Run a piece of your code without managing any server |
| IAM | Control who is allowed to do what |
| VPC | Build your own private network |

You'll learn exactly these, plus a few more, in this course.

---

## Part 2: Networking basics

### 2.1 IP addresses

Every device on a network has an **IP address**, which works like a phone number for computers. The common
format (**IPv4**) is four numbers from 0 to 255 separated by dots:

```
54.210.13.7
```

There are two kinds you need to know:

- **Public IP address**: reachable from anywhere on the internet. Like a phone number anyone can call.
- **Private IP address**: only works inside one private network (your home Wi-Fi, a company
  network, or an AWS network). These always come from reserved ranges, and the one you'll use
  most is `10.x.x.x`. Your home router probably gives your laptop something like `192.168.1.23`.
  Nobody on the internet can reach a private address directly.

### 2.2 CIDR notation (you'll need this in Module 03)

When you create a network, you tell AWS which **range** of private IP addresses it may use. You write
ranges in **CIDR notation**: an address, a slash, and a number.

```
10.0.0.0/16
```

**How to read it:** an IPv4 address is 32 bits long. The number after the slash says how many of
those bits are **fixed**. The remaining bits are free to vary, and each free bit doubles the
number of addresses in the range.

| CIDR | Fixed bits | Free bits | Number of addresses | Range it covers |
|---|---|---|---|---|
| `10.0.0.0/16` | 16 | 16 | 2^16 = **65,536** | `10.0.0.0` to `10.0.255.255` |
| `10.0.1.0/24` | 24 | 8 | 2^8 = **256** | `10.0.1.0` to `10.0.1.255` |
| `10.0.1.0/28` | 28 | 4 | 2^4 = **16** | `10.0.1.0` to `10.0.1.15` |
| `203.0.113.5/32` | 32 | 0 | **1** | exactly `203.0.113.5` |
| `0.0.0.0/0` | 0 | 32 | **all of them** | "anywhere on the internet" |

The shortcut that covers most real cases: **/16 fixes the first two numbers, /24 fixes the first three,
/32 is one single address, /0 is everything.** So `10.0.1.0/24` means "every address that starts
with `10.0.1.`".

Two you'll see constantly:
- `0.0.0.0/0` means "any IP address", i.e. the whole internet.
- `x.x.x.x/32` means "only this one IP address", e.g. your laptop.

### 2.3 Ports

One server can run several programs that each accept network connections (a web server, a
database, a remote login service). A **port** is a number from 0 to 65535 that says *which
program* on that machine a connection is for. If the IP address is the building's street
address, the port is the apartment number.

| Port | Used by |
|---|---|
| 22 | SSH (remote command-line login to Linux) |
| 80 | HTTP (websites, unencrypted) |
| 443 | HTTPS (websites, encrypted) |
| 3306 | MySQL database |
| 5432 | PostgreSQL database |

So "allow port 80 from `0.0.0.0/0`" means "let anyone on the internet connect to the web server on this machine."

### 2.4 TCP, HTTP, and HTTPS

- **TCP** is the underlying protocol that delivers data reliably between two machines. When
  AWS asks for a "protocol" in a firewall rule, you'll almost always pick TCP.
- **HTTP** is the language web browsers and web servers speak on top of TCP. A request has a
  **method** (what you want to do) and a **path** (which thing):
  - `GET /notes`: "give me the notes"
  - `POST /notes`: "create a new note" (the new note's data goes in the request **body**)
  - `DELETE /notes/42`: "delete note 42"
- The response has a **status code**: `200` OK, `201` created, `204` done with nothing to return,
  `403` forbidden, `404` not found, `500` the server crashed, `502`/`503`/`504` a server or load
  balancer in the middle couldn't get a good answer.
- **HTTPS** is HTTP encrypted with **TLS**, which needs a **certificate** that proves the
  server's identity.

### 2.5 DNS

Humans use names (`example.com`); computers use IP addresses. **DNS** is the internet's phone
book: it turns a name into an IP address. When AWS gives you something like
`lab-web-alb-123456.us-east-1.elb.amazonaws.com`, that's a DNS name. Your browser looks up its
IP automatically.

### 2.6 Firewalls

A **firewall** is a set of rules that decides which network connections are allowed. A rule
usually says: *allow* this **protocol** (TCP), on this **port** (80), **from** this source
(`0.0.0.0/0`). Anything not allowed is blocked. When a firewall blocks a connection, the
client usually doesn't get an error message; it just waits until it gives up. This is called
a **timeout**, and **a timeout almost always means a firewall or routing problem.** Remember
that; it will save you hours.

---

## Part 3: Data formats

### 3.1 JSON

AWS uses **JSON** everywhere: permission rules, command output, event data. JSON is a text format
for structured data. It has only a few building blocks:

```json
{
  "name": "Alice",
  "age": 31,
  "isAdmin": false,
  "skills": ["linux", "python"],
  "address": { "city": "Toronto", "country": "CA" }
}
```

- `{ }` is an **object**: a set of `"key": value` pairs, separated by commas.
- `[ ]` is an **array** (list): values separated by commas.
- Values can be text in double quotes (`"Alice"`), numbers (`31`), `true`/`false`, `null`,
  another object, or an array.
- **Keys must be in double quotes.** No comma after the last item. These two rules cause most JSON errors.

### 3.2 YAML

**YAML** holds the same kind of data as JSON but uses indentation instead of braces. You'll use it
in Modules 07 and 08 for infrastructure templates.

```yaml
name: Alice
age: 31
skills:
  - linux
  - python
address:
  city: Toronto
  country: CA
```

**Indentation is meaningful in YAML** (use spaces, never tabs). `address:` followed by indented
lines means those lines belong inside `address`.

---

## Part 4: The terminal (command line)

Most of this course happens in a **terminal**: a text window where you type a command, press
Enter, and the computer prints the result. The program that reads your commands is called the
**shell** (ours is **bash**). In Module 01 you'll open a terminal that runs inside your web
browser, so you don't need to install anything.

### 4.1 Anatomy of a command

```bash
aws s3 ls --region us-east-1
```

- `aws` is the **program** to run.
- `s3 ls` are **arguments** (here: which AWS service, and which action).
- `--region us-east-1` is an **option** (also called a **flag**): a name starting with `--`, followed by its value.

Rules:
- Words are separated by **spaces**. If a value itself contains spaces, wrap it in quotes: `"my file.txt"`.
- Capital letters matter. `ls` and `LS` are different.
- Nothing happens until you press **Enter**.

### 4.2 Every symbol used in this course

| You'll see | What it means | Example |
|---|---|---|
| `# text` | A **comment**. The shell ignores everything after `#`. We use comments to explain lines. | `ls   # list files` |
| `NAME=value` | Create a **variable** (a named box holding text). **No spaces around `=`.** | `BUCKET=my-bucket` |
| `$NAME` | Use the variable's value. The shell swaps `$BUCKET` for `my-bucket` before running the command. | `echo $BUCKET` |
| `export NAME=value` | Create a variable that programs you start (like `aws`) can also see. | `export AWS_REGION=us-east-1` |
| `$(command)` | Run the command inside, and put its output right here. | `ID=$(aws sts get-caller-identity --query Account --output text)` stores your account number in `ID` |
| `\` at the end of a line | "This command continues on the next line." It lets long commands be readable. **Nothing may come after the `\`, not even a space.** | see below |
| `a \| b` | A **pipe**: send the output of command `a` into command `b`. | `cat file.txt \| grep error` |
| `> file` | Write output into a file (replacing it). | `echo hello > note.txt` |
| `>> file` | Append output to the end of a file. | `echo more >> note.txt` |
| `cat > file <<'EOF'` ... `EOF` | A **heredoc**: everything between the first line and the line that says only `EOF` is written into `file`. We use this to create files without needing an editor. | see below |
| `'single quotes'` | Text is taken literally; `$NAME` is **not** replaced. | `echo '$HOME'` prints `$HOME` |
| `"double quotes"` | Text is kept together, but `$NAME` **is** replaced. | `echo "$HOME"` prints `/home/...` |
| `&&` | Run the second command only if the first succeeded. | `mkdir labs && cd labs` |
| `for x in a b c; do ...; done` | Repeat a command for each item. | `for i in 1 2 3; do echo $i; done` |
| `sleep 10` | Wait 10 seconds. | |

A multi-line command using `\`:

```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query Vpc.VpcId \
  --output text
```
This is exactly the same as typing it all on one line.

A heredoc that creates a file:

```bash
cat > hello.txt <<'EOF'
Hello
this is line two
EOF
```
After this runs, `hello.txt` contains the two lines in the middle. When you paste a heredoc, paste
the **whole block including the final `EOF` line**. If you forget it, the terminal shows `>` and
waits for more input. Type `EOF` and press Enter to finish.

### 4.3 Basic commands you'll use

| Command | What it does |
|---|---|
| `pwd` | Print which folder you're in |
| `ls` | List the files in the current folder (`ls -l` for details) |
| `mkdir labs` | Make a folder named `labs` |
| `cd labs` | Move into the folder `labs` (`cd ~` goes back to your home folder) |
| `cat file.txt` | Print a file's contents |
| `echo hello` | Print text (`echo $NAME` prints a variable) |
| `rm file.txt` | Delete a file (there's no trash bin) |
| `curl http://example.com` | Make an HTTP request from the terminal and print the response |
| `clear` | Clear the screen |

### 4.4 Keys that save you

- **Up arrow**: bring back the previous command.
- **Ctrl + C**: stop the command that's running right now (e.g. something that's "following" logs forever).
- **Tab**: auto-complete file and folder names.
- **Copy/paste** in a browser terminal: `Ctrl+C`/`Ctrl+V` on Windows/Linux, `Cmd+C`/`Cmd+V` on
  Mac. If pasting doesn't work, try `Ctrl+Shift+V` or right-click → Paste.

---

## Part 5: How we'll talk to AWS

Everything you do in AWS is a request to an **API**, which is a set of commands a program
accepts over the internet. There are several ways to send those requests, and they all do
the same thing:

1. **The AWS Management Console**: the website at `console.aws.amazon.com`. You click
   buttons, and the page sends API requests for you. Good for looking around.
2. **The AWS CLI (Command Line Interface)**: the `aws` command in a terminal. Precise,
   repeatable, and easy to copy from a course. **Most labs use this.**
3. **SDKs**: libraries for Python, JavaScript, and other languages, so your own programs can call AWS.
4. **Infrastructure as Code** (CloudFormation, Terraform): you describe everything in a file,
   and a tool makes the API calls. (Modules 07 and 08.)

### 5.1 Reading an AWS CLI command

```bash
aws ec2 describe-instances --query 'Reservations[].Instances[].InstanceId' --output text
```

- `aws`: the CLI program
- `ec2`: which **service**
- `describe-instances`: which **operation** (action). They're named `verb-noun`: `create-...`,
  `describe-...` (list/show), `delete-...`, `put-...` (write), `get-...` (read one thing).
- `--query '...'`: pick out only part of the response (explained below)
- `--output text`: print plain text instead of JSON. Options are `json` (default), `text`, `table`.

### 5.2 `--query`: picking out the part you need

AWS responses are often long JSON documents. `--query` filters them using a small language
called **JMESPath**. You only need three patterns:

Given this response:
```json
{ "Vpc": { "VpcId": "vpc-0abc123", "CidrBlock": "10.0.0.0/16" } }
```
- `--query Vpc.VpcId` returns `vpc-0abc123`. **Dots walk into objects.**

Given this response:
```json
{ "Buckets": [ { "Name": "photos" }, { "Name": "logs" } ] }
```
- `--query 'Buckets[].Name'` returns `photos` and `logs`. **`[]` means "for each item in the list".**
- `--query 'Buckets[0].Name'` returns `photos`. **`[0]` means "the first item".**

We combine `--query ... --output text` with `$( )` constantly, to save an ID into a variable:

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
```
Read it right to left: create a VPC, take just the ID from the response as plain text, and store it in `VPC_ID`.

### 5.3 ARNs: AWS's names for things

Every resource in AWS has an **ARN (Amazon Resource Name)**, a unique ID in this format:

```
arn:aws:<service>:<region>:<account-id>:<resource>
```
Examples:
- `arn:aws:iam::123456789012:role/lab-web-server-role`: an IAM role. IAM isn't tied to a region, so that part is blank.
- `arn:aws:s3:::my-bucket`: an S3 bucket. The region and account are blank because bucket names are globally unique.
- `arn:aws:lambda:us-east-1:123456789012:function:lab-order-worker`: a Lambda function.

You'll copy ARNs into permission rules to say exactly *which* resource a rule applies to.

---

## Checkpoint

Answer these before moving on. Answers are below.

1. How many IP addresses are in `10.0.5.0/24`? What's the first and last?
2. What does a firewall rule "TCP 443 from `0.0.0.0/0`" allow?
3. You run `NAME = alice` and get an error. Why?
4. What does this print: `X=world; echo "hello $X"`? And `echo 'hello $X'`?
5. You try to open a website and the browser spins for 30 seconds, then says "timed out." What's the most likely kind of problem?

<details><summary>Answers</summary>

1. 256 addresses, from `10.0.5.0` to `10.0.5.255`.
2. Anyone on the internet can open an HTTPS connection to that machine.
3. Spaces around `=`. The shell thinks `NAME` is a command. It must be `NAME=alice`.
4. `hello world`, then `hello $X` (single quotes don't substitute variables).
5. A firewall is silently dropping the connection, or there's no network route to the server.
</details>

---

**Next: [Module 01 · Account Setup](01-account-setup.md).** You'll create an AWS account, lock it down, and open your first terminal in AWS.
