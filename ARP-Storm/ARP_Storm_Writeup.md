# 🌪️ ARP Storm – Write-up

**Platform:** CyberTalents  
**Category:** Network Security  
**Difficulty:** Easy  

---

## 📝 Challenge Description

An attacker in the network is trying to poison the ARP table.

---

## 🔍 Initial Analysis

The challenge provided a **PCAP file**, so I began by analyzing it using `tshark` to quickly inspect the traffic.

```bash
tshark -r ARP+Storm.pcap
```

I immediately noticed repeated ARP packets with **unknown opcodes**:

```
ARP 42 Unknown ARP opcode 0x005a
ARP 42 Unknown ARP opcode 0x006d
ARP 42 Unknown ARP opcode 0x0078
...
```

Normally, ARP only uses:

| Opcode | Meaning      |
|--------|-------------|
| 1      | ARP Request |
| 2      | ARP Reply   |

Seeing many different **non-standard opcodes** strongly suggests that **data is being hidden inside the ARP opcode field**.

---

## 🧠 Step 1 – Extract the Opcode Values

```bash
tshark -r ARP+Storm.pcap | cut -d " " -f19,20 > codes.txt
```

Remove the `0x` prefix:

```bash
cat codes.txt | cut -d x -f2
```

---

## 🔄 Step 2 – Convert Hex to ASCII

```bash
cat codes.txt | cut -d x -f2 | xxd -r -p
```

Output:

```
ZmxhZ3tnckB0dWl0MHVzXzBwY09kZV8xc19BbHdAeXNfQTZ1U2VkX3QwX3AwMXMwbn0=
```

This clearly looks like **Base64 encoded data**.

---

## 🔓 Step 3 – Decode Base64

```bash
echo "ZmxhZ3tnckB0dWl0MHVzXzBwY09kZV8xc19BbHdAeXNfQTZ1U2VkX3QwX3AwMXMwbn0=" | base64 -d
```

Output:

```
flag{gr@tuit0us_0pcOde_1s_Alw@ys_A6uSed_t0_p01s0n}
```

---

## 🚩 Final Flag

```
flag{gr@tuit0us_0pcOde_1s_Alw@ys_A6uSed_t0_p01s0n}
```

---

## 🧠 Lessons Learned

- Attackers can hide data inside **non-standard protocol fields**
- ARP opcode fields can be abused as a **covert communication channel**
- When you see repeated **“Unknown” protocol values**, always suspect encoding or data exfiltration
- `tshark` combined with simple CLI tools is extremely powerful in CTF network forensics
