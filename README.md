[Zyxel-EX5601-T1-SSH-Unlock-IT.md](https://github.com/user-attachments/files/34871108/Zyxel-EX5601-T1-SSH-Unlock-IT.md)
# [GUIDA] Sbloccare SSH e SP Domain Nascosto su Zyxel EX5601-T1 (Odido) tramite Decifratura API

Se possiedi un router Zyxel EX5601-T1 (noto come T-56) fornito dal tuo ISP (come Odido) e stai cercando di abilitare l'SSH, avrai probabilmente letto guide online che dicono di andare su **Maintenance -> SP Domain** e aggiungere la tua LAN locale alla lista **SP Trust Domain**.

Tuttavia, sui firmware customizzati più recenti, **questo menu è completamente nascosto**.

In questa guida spiegherò esattamente perché non lo vedi, come l'API interna del router nasconda i dati usando la cifratura AES e come siamo riusciti ad aggirare la GUI nascosta per iniettare l'IP della nostra LAN locale nel Trust Domain, ottenendo con successo l'accesso SSH come root/admin.

---

## ⚠️ Nota Importante sul Firmware V5.70(ACEA.0)T56C_b10_0410

Questo problema specifico — dove il menu **SP Domain** è completamente rimosso dall'interfaccia e l'accesso SSH è bloccato — è una diretta conseguenza delle restrizioni implementate nel firmware **`V5.70(ACEA.0)T56C_b10_0410`** (distribuito da Odido).

Mentre le versioni precedenti o i firmware Zyxel generici permettevano semplicemente di navigare in *Maintenance -> SP Domain* per configurare l'accesso remoto, nella build **`b10_0410`** l'ISP ha deliberatamente:
1. **Nascosto l'UI:** Il codice della pagina `SPDomain` è ancora fisicamente presente nel pacchetto `app.js`, ma viene nascosto dinamicamente dal menu di navigazione in base ai livelli di privilegio dell'account.
2. **Impostato una Subnet Remota Fissa:** Il parametro interno `sp_trust_domain` del router è stato impostato su `10.1.3.0/24` di default. Questo significa che il router si aspetta connessioni SSH solo dalla rete di gestione interna dell'ISP, rifiutando attivamente qualsiasi tentativo di connessione proveniente dalla tua rete locale standard (es. `192.168.1.x`).

Poiché fare il downgrade del firmware non è sempre possibile (o compromette la compatibilità con l'ISP), il metodo di decifratura dell'API spiegato di seguito è l'unico modo per aggirare le restrizioni della **`b10_0410`** e inserire il tuo IP locale nel Trust Domain senza aver bisogno del pulsante (mancante) nella GUI.

---

## 🔐 Comprendere l'API Zyxel e la Cifratura AES

Potresti pensare: *"Se la pagina è nascosta, manderò semplicemente una richiesta POST diretta all'API del router!"* 
Se provi a monitorare il traffico di rete o a interrogare l'endpoint (`/cgi-bin/DAL?oid=sp_trust_domain`), non vedrai un JSON in chiaro. Otterrai qualcosa del genere:

```json
{
  "content": "qvnJSmwSOoCnjVbYlruzo6TzXS+yU5VYfReN7BMj7...",
  "iv": "pA785wOCrV5qjjiSrg4mF7NumQNgN8FjP8tDovKviqU="
}
```

**Ecco come Zyxel ha blindato il sistema:**
1. **Cifratura AES-CBC:** Tutte le comunicazioni tra la Web GUI e il backend CGI sono cifrate usando **AES (Advanced Encryption Standard)** in modalità CBC con padding PKCS7.
2. **Chiavi Dinamiche:** La chiave di cifratura (`AesKey`) viene generata dinamicamente dopo il login avvenuto con successo e salvata localmente nel `localStorage` del browser.
3. **Vettore di Inizializzazione (IV):** Ogni richiesta include un `iv` codificato in Base64 per garantire l'unicità crittografica.
4. **Token CSRF:** Il router richiede anche un `zySessionKey` valido negli header HTTP.

Per modificare il Trust Domain nascosto, dobbiamo cifrare manualmente la nostra nuova configurazione usando gli esatti parametri AES del router e iniettarla tramite la console del browser.

---

## 🛠️ La Soluzione Passo Passo

Ecco come manipolare l'API usando i Developer Tools del tuo browser per aggiungere la tua LAN al Trust Domain.

### Passo 1: Accedi al Router
Effettua il login normale all'interfaccia web del tuo router (solitamente `http://192.168.1.1`). Lascia la pagina aperta.

### Passo 2: Leggi il Trust Domain Attuale (Nascosto)
Premi **F12** per aprire i Developer Tools, vai nella scheda **Console** ed esegui il seguente script. Questo script recupera i dati cifrati dall'endpoint nascosto `sp_trust_domain` e li decifra usando la chiave AES salvata nel tuo browser.

*(Nota: a seconda del browser, potresti dover digitare "allow pasting" prima che ti permetta di incollare il codice).*

```javascript
// Recupera la chiave AES dalla sessione attuale
var aesKey = localStorage.getItem("AesKey");

$.ajax({
    url: "/cgi-bin/DAL?oid=sp_trust_domain",
    method: "GET",
    success: function(response) {
        var parsedIv = CryptoJS.enc.Base64.parse(response.iv);
        var parsedKey = CryptoJS.enc.Base64.parse(aesKey);
        
        // Decifra il payload AES
        var decrypted = CryptoJS.AES.decrypt(response.content, parsedKey, {
            iv: parsedIv,
            mode: CryptoJS.mode.CBC,
            padding: CryptoJS.pad.Pkcs7
        }).toString(CryptoJS.enc.Utf8);
        
        console.log("Configurazione Trust Domain attuale:", JSON.parse(decrypted));
    }
});
```
*Eseguendo questo comando, abbiamo scoperto che il router accettava connessioni solo da `10.1.3.0/24`!*

### Passo 3: Inietta il Nuovo Trust Domain
Ora, dobbiamo sovrascrivere quella subnet `10.x` con la nostra rete locale (`192.168.1.0/24`). Incolla il seguente codice nella console e premi Invio. Cifrerà il nuovo payload JSON e lo invierà al router.

```javascript
// 1. Definisci la nostra nuova subnet LAN fidata
var nuovoOggetto = {
    "Index": 1,
    "IPAddressPrefix": "192.168.1.0/24" // Cambia questo valore se il tuo router usa una subnet diversa (es. 192.168.0.0/24)
};

// 2. Prepara Chiavi di Cifratura e IV
var key = localStorage.getItem("AesKey");
var ivBytes = CryptoJS.lib.WordArray.random(32).toString(CryptoJS.enc.Base64);
var ivParsed = CryptoJS.enc.Base64.parse(ivBytes);
var keyParsed = CryptoJS.enc.Base64.parse(key);

// 3. Cifra il payload usando AES-CBC
var payloadString = JSON.stringify(nuovoOggetto);
var encrypted = CryptoJS.AES.encrypt(payloadString, keyParsed, {
    iv: ivParsed,
    mode: CryptoJS.mode.CBC,
    padding: CryptoJS.pad.Pkcs7
}).toString();

// 4. Invia la richiesta POST cifrata al router
$.ajax({
    method: "POST", 
    url: "/cgi-bin/DAL?oid=sp_trust_domain",
    dataType: "json",
    contentType: "application/json",
    headers: {
        "CSRFToken": localStorage.getItem("zySessionKey")
    },
    data: JSON.stringify({
        content: encrypted,
        iv: ivBytes
    }),
    success: function(response) {
        console.log("Risposta dal router:", response);
    },
    error: function(err) {
        console.error("Errore durante l'iniezione:", err);
    }
});
```

Se andato a buon fine, la console stamperà una risposta cifrata. Decifrandola si otterrà:
`{ result: "ZCFG_SUCCESS" }`

---

## 🎯 Il Risultato Finale

Una volta ricevuto il messaggio di successo, il router accetterà immediatamente le connessioni SSH dalla tua LAN locale.

Apri il terminale (Linux/macOS) o PuTTY (Windows) e connettiti:
`ssh admin@192.168.1.1`

Inserisci la password di admin, e sei dentro! Ecco il vero dump del terminale che dimostra l'accesso riuscito sull'Odido EX5601-T1:

```text
└╼ale$ssh admin@192.168.1.1
admin@192.168.1.1's password: 
ZySH> sys atsh
Firmware Version          : V5.70(ACEA.0)T56C_b10_0410
Bootbase Version          : V3.2 | 03/07/2025 11:51:27
Vendor Name               : ZYXEL Communication Corp
Product Model             : EX5601-T1
Serial Number             : S240Y28010085
First MAC Address         : 909F2204D58B
Last MAC Address          : 909F2204D595
MAC Address Quantity      : 11
Default Country Code      : 00
Boot Module Debug Flag    : 00
Kernel Checksum           : 00003BC7
RootFS Checksum           : 0000535E
Romfile Checksum          : 0000E73A
Main Feature Bits         : 00
Other Feature Bits        : 
c2f7f0e2: 040a000b ffffffff 00000000 00000000
c2f7f0f2: ffffffff ffffffff ffffffff
ZySH> sys atwz 909F2204D58B 0 1
MAC address  : 90:9F:22:04:D5:8B
Country Code : 00
FeatureBit   : 6F
MAC Number   : AC
ZySH> sys atck
supervisor password: FUErjam8LH
admin password     : QHB8A4D8
WiFi PSK key       : KLMQV3XRLHB68SYB
ZySH> 
```

**Goditi il tuo router sbloccato!** 
Utilizzando la libreria nativa `CryptoJS` del browser, abbiamo completamente aggirato le limitazioni della GUI, interagito con il backend CGI cifrato in AES di Zyxel, e sbloccato il demone SSH locale che era nascosto dietro un Trust Domain restrittivo di classe `10.x`.
