# 🕵️‍♂️ CTF Write-Up — Medium Network/Malware Challenge

## 🎯 Challenge Category
**Network Forensics + Malware Analysis + Cryptography**

The goal of this challenge was to recover exfiltrated data sent from an infected host by analyzing network traffic and reverse-engineering the malware’s encryption routine.

---

## 📦 Step 1 — Inspecting the PCAP

We opened the provided capture file **`Medium_Chall.pcapng`** in Wireshark and applied the filter:

```
http
```

We noticed suspicious communication between:

| Role      | IP Address      |
|-----------|-----------------|
| Victim    | `192.168.1.4`    |
| Attacker  | `192.168.1.11:7788` |

---

## 🚨 Malicious HTTP Request

One of the most important findings was the following HTTP POST request:

```
POST /result/razer HTTP/1.1
User-Agent: Mozilla/5.0 ... WindowsPowerShell/5.1
Content-Type: application/x-www-form-urlencoded
```

The presence of a **PowerShell User-Agent** strongly indicates that a malicious PowerShell script was running on the victim machine.

The exfiltrated data looked like:

```
result=FCvJ06GsQNacpIeJtKbaet5y2nSKUY0...
```

This clearly appeared to be **Base64-encoded encrypted data**.

---

## 🧠 Step 2 — Recovering the Malware Script

Further analysis of the traffic revealed that the victim downloaded a PowerShell script:

```
GET /down/razer/host.ps1
```

Using **Follow HTTP Stream** in Wireshark, we reconstructed the full PowerShell malware script.

---

## 🔐 Step 3 — Understanding the Encryption

Inside the script, we found the encryption setup:

```powershell
function Create-AesManagedObject($key, $IV) {
    $aesManaged = New-Object "System.Security.Cryptography.AesManaged"
    $aesManaged.Mode = [System.Security.Cryptography.CipherMode]::CBC
    $aesManaged.Padding = [System.Security.Cryptography.PaddingMode]::Zeros
    $aesManaged.BlockSize = 128
    $aesManaged.KeySize = 256
}
```

This tells us the malware uses:

> **AES-256 in CBC mode with Zero Padding**

---

### 🔒 Encryption Logic

```powershell
function Encrypt-String($key, $unencryptedString) {
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($unencryptedString)
    $aesManaged = Create-AesManagedObject $key
    $encryptor = $aesManaged.CreateEncryptor()
    $encryptedData = $encryptor.TransformFinalBlock($bytes, 0, $bytes.Length)
    $fullData = $aesManaged.IV + $encryptedData
    [System.Convert]::ToBase64String($fullData)
}
```

Key observation:

> The **IV is prepended** to the ciphertext before Base64 encoding.

---

### 🔓 Decryption Logic

```powershell
function Decrypt-String($key, $encryptedStringWithIV) {
    $bytes = [System.Convert]::FromBase64String($encryptedStringWithIV)
    $IV = $bytes[0..15]
    $aesManaged = Create-AesManagedObject $key $IV
    $decryptor = $aesManaged.CreateDecryptor()
}
```

To decrypt, we must:
1. Base64 decode the data
2. Extract the first 16 bytes as the IV
3. Use the remaining bytes as ciphertext
4. Decrypt using AES-CBC with the extracted IV

---

## 🔑 Step 4 — Extracting the AES Key

At the bottom of the script we found:

```powershell
$key = "llm0xBW0fV9ssq+f0sIMFK6Qy0H0zhdenMRziNqXA="
```

This is a **Base64-encoded AES key**.

---

## 🧪 Step 5 — Decrypting the Exfiltrated Data

### Decryption Process

1. Copy the value after `result=` from the PCAP
2. URL-decode it (replace `%3D` with `=` if present)
3. Base64-decode the data
4. Extract IV (first 16 bytes)
5. Decrypt the rest using AES-256-CBC with the recovered key

---

### 🐍 Python Decryption Script

```python
import base64
from Crypto.Cipher import AES

key_b64 = "llm0xBW0fV9ssq+f0sIMFK6Qy0H0zhdenMRziNqXA="
key = base64.b64decode(key_b64)

data_b64 = "PUT_EXFILTRATED_RESULT_HERE"
data = base64.b64decode(data_b64)

iv = data[:16]
ciphertext = data[16:]

cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = cipher.decrypt(ciphertext)

print(plaintext.rstrip(b"\x00").decode())
```

---

## 🏁 Final Result

After decrypting the captured payload, we successfully recovered the **exfiltrated data**, which contained the challenge flag.

---

## 🧩 Skills Demonstrated

| Stage | Skill |
|------|------|
| PCAP analysis | Network Forensics |
| Script extraction | Traffic Reconstruction |
| PowerShell review | Malware Analysis |
| AES analysis | Cryptography |
| Writing decryption code | Reverse Engineering |

---

## 🔥 Conclusion

The attacker:
1. Delivered a malicious PowerShell script  
2. Used AES-256-CBC encryption  
3. Prepended the IV to ciphertext  
4. Base64-encoded the final payload before exfiltration  

By combining network forensics and reverse engineering, we were able to reconstruct the encryption logic and successfully decrypt the stolen data.
