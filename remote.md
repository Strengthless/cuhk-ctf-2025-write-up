# CUHK CTF 2025 Writeup - The Betrayal - Remote Intrusion

written by Johnnnny

### Challenge Description

It seems like an attacker is attempting to remotely access the computer of the CEO of Icey Penguin Marketing Agency. The Incident Response team has extracted the login events for you. However, since remoting into the device using RDP is quite common in Icey Penguin Marketing Agency, it is quite hard to trace down the attack. Can you discover the attacker's identity?

---

### Step 1: Convert the Event Log to JSON File

found a tool online: [EVTX Viewer](https://omerbenamram.github.io/evtx/)

Import the EVTX file and export a JSON file.

---

### Step 2: Identify Suspicious Field

searched for the keyword `%%2313` which is the FailureReason code - Unknown user name or bad password.

found 18 records with target name `WXpnbCBXbGx4IG5mc3YyNW5lcXtoVHlfcEczeUVfdzBSX2RTMGhfMVpydFlfRjUzY19UeV9hVGx0WWUzaUVfbzB3ZkkyRn0=`

### Step 3: Base64 Decode

Decoding with `base64`:

Decode `WXpnbCBXbGx4IG5mc3YyNW5lcXtoVHlfcEczeUVfdzBSX2RTMGhfMVpydFlfRjUzY19UeV9hVGx0WWUzaUVfbzB3ZkkyRn0=` with `base64` using CyberChef

Output:

```
Yzgl Wllx nfsv25neq{hTy_pG3yE_w0R_dS0h_1ZrtY_F53c_Ty_aTltYe3iE_o0wfI2F}
```

Now we see a **flag-like format** with `{}`, but still encoded/obfuscated.

---

### Step 4: Apply ROT13 Brute Force

The result shows strings like `"nfsv25neq"` which look like ROT13 candidates.

Try ROT13 brute force using CyberChef:

Output:

```
Amount = 14: Mnuz Kzzl btgj25bse{vHm_dU3mS_k0F_rG0v_1NfhM_T53q_Hm_oHzhMs3wS_c0ktW2T}
Amount = 15: Nova Laam cuhk25ctf{wIn_eV3nT_l0G_sH0w_1OgiN_U53r_In_pIaiNt3xT_d0luX2U}
Amount = 16: Opwb Mbbn dvil25dug{xJo_fW3oU_m0H_tI0x_1PhjO_V53s_Jo_qJbjOu3yU_e0mvY2V}
```

---

### Step 5: Final Flag

After base64 + ROT13 brute force, the **flag is revealed**:

```
cuhk25ctf{wIn_eV3nT_l0G_sH0w_1OgiN_U53r_In_pIaiNt3xT_d0luX2U}
```

---

### Tools Used

-   CyberChef for `base64` & `ROT13` processing

---
