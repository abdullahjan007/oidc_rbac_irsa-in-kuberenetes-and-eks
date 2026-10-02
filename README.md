# oidc_rbac_irsa-in-kuberenetes-and-eks

# EKS ServiceAccount, RBAC, OIDC & IRSA — Roman Urdu Notes

> Ye notes EKS mein ServiceAccount, RBAC, OIDC, IAM Role aur IRSA ko simple Roman Urdu mein samajhne aur baad mein revise karne ke liye hain.

---

# Table of Contents

1. [Sab se Pehle Basic Concept](#sab-se-pehle-basic-concept)
2. [Kubernetes ServiceAccount Kya Hai?](#kubernetes-serviceaccount-kya-hai)
3. [ServiceAccount Kyun Chahiye?](#serviceaccount-kyun-chahiye)
4. [ServiceAccount = WHO ARE YOU?](#serviceaccount--who-are-you)
5. [RBAC = WHAT CAN YOU DO?](#rbac--what-can-you-do)
6. [ServiceAccount + RBAC](#serviceaccount--rbac)
7. [Ab AWS Wala Masla](#ab-aws-wala-masla)
8. [OIDC Kya Hai?](#oidc-kya-hai)
9. [OIDC Provider URL](#oidc-provider-url)
10. [OIDC URL vs IAM OIDC Provider](#oidc-url-vs-iam-oidc-provider)
11. [IAM Role Kya Hai?](#iam-role-kya-hai)
12. [Trust Policy vs Permission Policy](#trust-policy-vs-permission-policy)
13. [AWS STS Kya Karta Hai?](#aws-sts-kya-karta-hai)
14. [IRSA Kya Hai?](#irsa-kya-hai)
15. [Complete Practical Flow](#complete-practical-flow)
16. [Real Life Analogy](#real-life-analogy)
17. [Sab Concepts Ek Sath](#sab-concepts-ek-sath)
18. [Interview Questions](#interview-questions)
19. [Quick Revision](#quick-revision)

---

# Sab Se Pehle Basic Concept

Hum bilkul zero se start karte hain.

Suppose hamara EKS cluster hai:

```text
EKS Cluster
    |
    └── Pod
         |
         └── my-app
```

Ab `my-app` Kubernetes API ko koi request bhejta hai.

Kubernetes naturally poochega:

> **"Bhai, tum ho kaun?"**

Yahan se **ServiceAccount** ka concept start hota hai.

---

# Kubernetes ServiceAccount Kya Hai?

ServiceAccount basically **application/Pod ki identity** hoti hai.

Simple words mein:

> **ServiceAccount Kubernetes ko batata hai ke request kis identity se aa rahi hai.**

Real-life example:

```text
Abdullah
   |
   ↓
Employee ID
```

Company ko Employee ID se pata chalta hai:

> "Ye banda kaun hai?"

Kubernetes mein:

```text
Application
   |
   ↓
ServiceAccount
```

Kubernetes ko pata chalta hai:

> "Ye request kis application/workload ki identity se aa rahi hai?"

---

# ServiceAccount = WHO ARE YOU?

Ye sab se important line hai:

```text
ServiceAccount = WHO ARE YOU?
```

Example:

```text
EKS
 |
 └── Pod
      |
      └── my-app
           |
           └── ServiceAccount
                my-app-sa
```

Yani Pod ko hum bol rahe hain:

> "Tumhari Kubernetes identity `my-app-sa` hai."

Pod YAML:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: my-app

spec:
  serviceAccountName: my-app-sa

  containers:
    - name: app
      image: nginx
```

Yahan ye line sab se important hai:

```yaml
serviceAccountName: my-app-sa
```

Iska matlab:

> **"Is Pod ki identity `my-app-sa` hai."**

---

# ServiceAccount Kyun Chahiye?

Ab maan lo hamare cluster mein 3 applications hain:

```text
Pod A → Payment App
Pod B → Monitoring App
Pod C → Backup App
```

Hum nahi chahte ke Kubernetes ke nazdeek teeno applications ek hi identity hon.

Hum separate identities bana sakte hain:

```text
Payment App
    ↓
payment-sa

Monitoring App
    ↓
monitoring-sa

Backup App
    ↓
backup-sa
```

Ab agar Kubernetes ko request mile:

```text
Request
   ↓
"Who are you?"
   ↓
monitoring-sa
```

Kubernetes ko pata hai:

> "Okay, request monitoring application ki identity se aa rahi hai."

Isi tarah:

```text
Request
   ↓
"Who are you?"
   ↓
payment-sa
```

Kubernetes ko pata hai:

> "Ye payment application ki identity hai."

---

# RBAC = WHAT CAN YOU DO?

Ab yahan ek important cheez hai.

Sirf ye pata chal jana ke:

> **"Tum kaun ho?"**

iska matlab ye nahi ke:

> **"Tum sab kuch kar sakte ho."**

Real life mein:

```text
Employee ID
    ↓
Who are you?
```

Lekin Employee ID hone ka matlab ye nahi ke tum:

```text
HR Database       ❌
Server Room       ❌
CEO Office        ❌
Company Data      ❌
```

access kar sakte ho.

Permissions alag cheez hain.

Kubernetes mein permissions ke liye **RBAC** use hota hai.

RBAC:

> **Role-Based Access Control**

Easy way:

```text
ServiceAccount
      ↓
WHO ARE YOU?

RBAC
      ↓
WHAT CAN YOU DO?
```

---

# ServiceAccount + RBAC

Suppose hamare paas:

```text
ServiceAccount
monitoring-sa
```

Hum chahte hain monitoring application:

```text
GET Pods       ✅
LIST Pods      ✅
WATCH Pods     ✅

DELETE Pods    ❌
```

Toh hum RBAC Role bana sakte hain:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: pod-reader

rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs:
      - get
      - list
      - watch
```

Phir RoleBinding ke through ServiceAccount ko Role ke saath connect karte hain.

Conceptually:

```text
ServiceAccount
monitoring-sa
      |
      ↓
RoleBinding
      |
      ↓
Role
pod-reader
      |
      ↓
GET/LIST/WATCH Pods
```

Toh:

```text
ServiceAccount
      ↓
WHO ARE YOU?
      ↓
monitoring-sa

RBAC
      ↓
WHAT CAN YOU DO?
      ↓
GET/LIST/WATCH Pods
```

---

# Ab AWS Wala Masla

Ab tak sab Kubernetes ke andar tha.

Ab EKS mein AWS bhi aa gaya.

Suppose:

```text
EKS
 |
 └── Pod
      |
      └── my-app
           |
           └── my-app-sa
```

Kubernetes jaanta hai:

```text
my-app-sa
```

Yani:

> "Ye application `my-app-sa` identity se aa rahi hai."

Ab maan lo `my-app` ko AWS S3 access karna hai:

```text
my-app
   ↓
S3
```

Yahan problem hai.

`my-app-sa` **Kubernetes ki identity** hai.

Lekin:

```text
S3
```

AWS ka resource hai.

AWS naturally ye nahi samajhta:

```text
my-app-sa
```

as an AWS identity.

AWS ki apni identity system hai:

```text
IAM Role
IAM Policy
```

Toh humein Kubernetes aur AWS ke darmiyan connection chahiye.

---

# OIDC Kya Hai?

OIDC ka full form hai:

> **OpenID Connect**

Isko filhaal simple words mein samjho:

> **OIDC AWS ko Kubernetes ki identity ko trust/verify karne ka mechanism provide karta hai.**

Flow:

```text
Kubernetes
    |
    ↓
ServiceAccount
    |
    ↓
OIDC Token
    |
    ↓
AWS
```

ServiceAccount ki identity token ke through represent hoti hai.

Conceptually token AWS ko bata sakta hai:

```text
"Main EKS cluster se hoon."

"Meray ServiceAccount ki identity:
system:serviceaccount:default:my-app-sa
hai."
```

AWS phir verify kar sakta hai ke token kis trusted issuer ne issue kiya.

---

# OIDC Provider URL

Har EKS cluster ka ek OIDC issuer URL hota hai.

Example:

```text
https://oidc.eks.us-east-1.amazonaws.com/id/ABC123XYZ
```

Ye URL EKS cluster ke **OIDC issuer** ko identify karta hai.

Tum AWS CLI se dekh sakte ho:

```bash
aws eks describe-cluster \
  --name my-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Example output:

```text
https://oidc.eks.us-east-1.amazonaws.com/id/ABC123XYZ
```

---

# OIDC Provider URL Ko Simple Tarike Se Samjho

Is URL ko ek **trusted identity source** samjho.

Conceptually:

```text
EKS Cluster
     |
     ↓
OIDC Issuer
     |
     ↓
"Issue/identify identities for this EKS cluster"
```

AWS IAM ko hum bata sakte hain:

> "Main is EKS cluster ke OIDC issuer par trust karta hoon."

So:

```text
EKS
 |
 └── OIDC Issuer URL
          |
          ↓
     IAM trusts it
```

---

# OIDC URL vs IAM OIDC Provider

Ye dono confuse mat karna.

## 1. OIDC Issuer URL

Example:

```text
https://oidc.eks.us-east-1.amazonaws.com/id/ABC123XYZ
```

Ye **EKS cluster ka issuer URL** hai.

---

## 2. IAM OIDC Identity Provider

AWS IAM mein hum is OIDC issuer ko register/trust karte hain.

Conceptually:

```text
EKS OIDC Issuer
       |
       ↓
IAM OIDC Identity Provider
       |
       ↓
AWS IAM trusts this issuer
```

IAM mein iska ARN kuch is tarah hota hai:

```text
arn:aws:iam::<ACCOUNT_ID>:oidc-provider/
oidc.eks.us-east-1.amazonaws.com/id/ABC123XYZ
```

Notice karo:

```text
https://
```

IAM ARN mein nahi hota.

---

# AWS Ko OIDC Ki Zaroorat Kyun Hai?

Suppose Pod AWS ko kehta hai:

```text
"Main my-app-sa hoon."
```

AWS simply ye nahi kahega:

> "Okay, maan liya." 😄

AWS ko verify karna hai:

> "Ye identity kisne issue ki?"

Token ke andar issuer information hoti hai.

Conceptually:

```text
Pod
 ↓
ServiceAccount
 ↓
OIDC Token
 ↓
AWS
 ↓
"Kis issuer ne issue kiya?"
 ↓
EKS OIDC Issuer
 ↓
"Is issuer par trust hai?"
 ↓
YES
```

Lekin sirf issuer trusted hona enough nahi.

IAM Role ki trust policy ye bhi specify kar sakti hai:

> "Main sirf is particular ServiceAccount ko trust karta hoon."

---

# IAM Role Kya Hai?

AWS mein IAM Role ko simple way mein samjho:

> **IAM Role AWS permissions provide karta hai.**

Example:

```text
IAM Role
my-app-s3-role
```

Is role ko permission di ja sakti hai:

```text
S3
 |
 └── GetObject
```

Yani:

```text
my-app-s3-role
       |
       ↓
S3 GetObject
```

---

# Trust Policy vs Permission Policy

Ye concept bohat important hai.

IAM Role ke context mein do alag questions hain.

## Trust Policy

Question:

> **WHO can assume this role?**

Example:

```text
EKS OIDC Provider
       ↓
my-app-sa
       ↓
Allowed to assume IAM Role
```

---

## Permission Policy

Question:

> **Role assume karne ke baad WHAT can it do?**

Example:

```text
IAM Role
   ↓
S3
   ↓
GetObject
```

So simple:

```text
Trust Policy
    ↓
WHO CAN ASSUME ME?

Permission Policy
    ↓
WHAT CAN I DO?
```

---

# AWS STS Kya Karta Hai?

STS:

> **AWS Security Token Service**

Iska basic kaam yahan ye hai ke successful role assumption ke baad **temporary AWS credentials** provide kare.

Flow:

```text
Pod
 ↓
OIDC Token
 ↓
AWS STS
 ↓
IAM Role Assume
 ↓
Temporary Credentials
 ↓
S3
```

Important:

Hum normally Pod ke andar permanent:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

rakhne ke bajaye temporary credentials ka mechanism use kar sakte hain.

---

# IRSA Kya Hai?

IRSA ka full form:

> **IAM Roles for Service Accounts**

Naam se hi idea mil raha hai:

```text
IAM Role
    +
ServiceAccount
```

IRSA ka purpose:

> **Kubernetes ServiceAccount ko AWS IAM Role ke saath associate karna taake Pod AWS resources ko IAM permissions ke according access kar sake.**

Simple:

```text
Kubernetes                         AWS

ServiceAccount  ←── IRSA ──→       IAM Role
                                      |
                                      ↓
                                     S3
```

---

# Complete Practical Flow

Ab saare concepts ek saath dekhte hain.

Suppose:

```text
EKS Cluster
    |
    └── payment-api Pod
```

Pod use karta hai:

```text
payment-api-sa
```

Aur application ko S3 read karna hai.

Complete flow:

```text
                    EKS
                     |
                     ↓
                    Pod
                     |
                     ↓
              ServiceAccount
               payment-api-sa
                     |
                     ↓
                OIDC Token
                     |
                     ↓
              EKS OIDC Issuer
                     |
                     ↓
                  AWS STS
                     |
                     ↓
              IAM Role
           payment-api-s3-role
                     |
                     ↓
           Temporary Credentials
                     |
                     ↓
                    S3
```

---

# Real Practical Example

## Step 1 — ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: payment-api-sa
```

Ab:

```text
payment-api
     ↓
payment-api-sa
```

Application ki Kubernetes identity set ho gayi.

---

## Step 2 — Pod

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: payment-api

spec:
  serviceAccountName: payment-api-sa

  containers:
    - name: payment-api
      image: payment-api:latest
```

Important:

```yaml
serviceAccountName: payment-api-sa
```

Ye Pod ko ServiceAccount ke saath associate karta hai.

---

## Step 3 — EKS OIDC

EKS cluster ka OIDC issuer:

```text
https://oidc.eks.us-east-1.amazonaws.com/id/ABC123XYZ
```

IAM mein is issuer ko trust kiya ja sakta hai.

---

## Step 4 — IAM Role

Create:

```text
payment-api-s3-role
```

Role ki permission:

```text
S3 → GetObject
```

Trust policy conceptually kahegi:

```text
Main is EKS OIDC provider par trust karta hoon.

Aur specifically:

system:serviceaccount:default:payment-api-sa
```

ko role assume karne ki permission hai.

````

---

# Step 5 — Application S3 Access Karti Hai

Ab complete flow:

```text
payment-api
      |
      ↓
payment-api-sa
      |
      ↓
OIDC Token
      |
      ↓
AWS STS
      |
      ↓
Trust Policy Check
      |
      ↓
"Trusted?"
      |
     YES
      |
      ↓
IAM Role
      |
      ↓
Temporary AWS Credentials
      |
      ↓
S3
````

---

# IRSA Ka Main Faida

Suppose EKS node par multiple Pods hain:

```text
Node 1
 |
 ├── payment-api
 ├── frontend
 └── monitoring
```

Agar node ka IAM Role bohat broad permissions rakhta ho:

```text
Node IAM Role
     |
     ↓
S3 Full Access
```

toh unnecessarily bohat saare workloads ko broad AWS permissions mil sakti hain.

IRSA mein hum workload-specific identity use kar sakte hain:

```text
payment-api
    ↓
payment-api-sa
    ↓
payment-api-s3-role
    ↓
S3 Read
```

Aur doosri application:

```text
monitoring
    ↓
monitoring-sa
    ↓
monitoring-role
    ↓
Different permissions
```

Yani AWS permissions workload ke level par zyada specifically control ki ja sakti hain.

---

# Sab Concepts Ek Sath

Ab is diagram ko yaad kar lo:

```text
                 KUBERNETES
                     |
                     ↓
                    Pod
                     |
                     ↓
              ServiceAccount
                     |
                "WHO ARE YOU?"
                     |
                     ↓
                   RBAC
                     |
          "Kubernetes mein
           kya kar sakte ho?"
                     |
                     ↓
                   OIDC
                     |
          "AWS, is identity
           ko trust/verify karo"
                     |
                     ↓
                 AWS STS
                     |
                     ↓
                 IAM Role
                     |
            "AWS mein kya
             kar sakte ho?"
                     |
                     ↓
              AWS Services
             /      |       \
            S3     ECR    Secrets Manager
```

---

# Sab Se Important Difference

Ye 6 lines yaad kar lo:

```text
ServiceAccount
→ WHO ARE YOU?

RBAC
→ Kubernetes mein WHAT CAN YOU DO?

OIDC
→ AWS ko Kubernetes identity par trust/identity federation ka mechanism

IAM Role
→ AWS mein WHAT CAN YOU DO?

Trust Policy
→ WHO CAN ASSUME THIS IAM ROLE?

Permission Policy
→ IAM Role AWS mein WHAT CAN DO?
```

---

# Ek Simple Real-Life Analogy

Company ka example:

### 1. Employee ID

```text
Employee
   ↓
Employee ID
```

Question:

> Tum kaun ho?

Kubernetes equivalent:

```text
ServiceAccount
```

---

### 2. Company Access Rules

```text
Employee ID
   ↓
Access Rules
```

Question:

> Tum company mein kya access kar sakte ho?

Kubernetes equivalent:

```text
RBAC
```

---

### 3. External Organization Verification

AWS doosri organization samjho.

AWS ko verify karna hai:

> "Kya ye identity trusted Kubernetes/EKS source se aayi hai?"

Yahan:

```text
OIDC
```

identity federation/trust ka mechanism provide karta hai.

---

### 4. AWS Access

AWS mein actual permissions:

```text
IAM Role
    ↓
IAM Policy
    ↓
S3 / ECR / Secrets Manager
```

---

# Interview Questions

## Q1: Kubernetes ServiceAccount kya hota hai?

**Answer:**

> ServiceAccount Kubernetes workload ki identity hoti hai. Iske through Kubernetes identify kar sakta hai ke request kis application/workload ki identity se aa rahi hai.

---

## Q2: ServiceAccount kyun use karte hain?

**Answer:**

> ServiceAccount workloads ko unique Kubernetes identity dene ke liye use hota hai taake different applications ki requests ko identify aur unke permissions ko control kiya ja sake.

---

## Q3: ServiceAccount aur RBAC mein kya difference hai?

```text
ServiceAccount
→ WHO ARE YOU?

RBAC
→ WHAT CAN YOU DO?
```

---

## Q4: OIDC kya hai?

**Answer:**

> OIDC OpenID Connect hai. EKS mein ye identity federation ka mechanism provide karta hai jiske through AWS IAM EKS ke OIDC issuer se aane wali workload identity ko trust/verify kar sakta hai.

---

## Q5: EKS OIDC Provider URL kya hota hai?

**Answer:**

> Ye EKS cluster ka OIDC issuer URL hota hai jo cluster ke identity issuer ko identify karta hai. IAM is issuer ko trust kar sakta hai, jo IRSA ke through Kubernetes ServiceAccount ko AWS IAM Role ke saath connect karne mein use hota hai.

Example:

```text
https://oidc.eks.us-east-1.amazonaws.com/id/ABC123XYZ
```

---

## Q6: IRSA kya hai?

**Answer:**

> IRSA ka matlab IAM Roles for Service Accounts hai. Iske through Kubernetes ServiceAccount ko AWS IAM Role ke saath associate kiya jata hai taake us ServiceAccount ko use karne wale Pods AWS resources ko IAM permissions ke mutabiq access kar saken.

---

## Q7: IAM Trust Policy kya decide karti hai?

**Answer:**

> Trust Policy decide karti hai ke kaun IAM Role ko assume kar sakta hai.

Simple:

```text
Trust Policy
→ WHO CAN ASSUME ME?
```

---

## Q8: IAM Permission Policy kya decide karti hai?

**Answer:**

> Permission Policy decide karti hai ke IAM Role assume karne ke baad AWS resources par kya actions perform kiye ja sakte hain.

Simple:

```text
Permission Policy
→ WHAT CAN I DO?
```

---

# Quick Revision

Agar tum sab bhool jao to sirf ye yaad karo:

```text
ServiceAccount
=
WHO ARE YOU?

RBAC
=
WHAT CAN YOU DO IN KUBERNETES?

OIDC
=
AWS KO KUBERNETES IDENTITY PAR TRUST/VERIFY KARNE KA MECHANISM

IAM Role
=
WHAT CAN YOU DO IN AWS?

Trust Policy
=
WHO CAN ASSUME THE IAM ROLE?

Permission Policy
=
WHAT CAN THE IAM ROLE DO?
```

Aur complete flow:

```text
Pod
 ↓
ServiceAccount
 ↓
OIDC Token
 ↓
EKS OIDC Issuer
 ↓
AWS STS
 ↓
IAM Role
 ↓
Temporary AWS Credentials
 ↓
AWS Service
```

---

# Final Mental Model

Sab se simple tareeqe se:

```text
ServiceAccount
        ↓
"Main kaun hoon?"
        ↓
      WHO?

RBAC
        ↓
"Main Kubernetes mein kya kar sakta hoon?"
        ↓
      WHAT?

OIDC
        ↓
"AWS, meri Kubernetes identity ko trust/verify karo"
        ↓
      TRUST

IAM Role
        ↓
"Main AWS mein kya kar sakta hoon?"
        ↓
      AWS PERMISSIONS
```

**IRSA = Kubernetes ServiceAccount ko AWS IAM Role ke saath connect karna.**

```text
Kubernetes                         AWS

Pod
 ↓
ServiceAccount
 ↓
    └──────── IRSA / OIDC ───────→ IAM Role
                                      ↓
                                      S3
```

> **Sab se important line:**
> **ServiceAccount = WHO ARE YOU?**
> **RBAC = Kubernetes mein WHAT CAN YOU DO?**
> **OIDC = AWS ko identity par trust/federation ka mechanism**
> **IAM Role = AWS mein WHAT CAN YOU DO?**
