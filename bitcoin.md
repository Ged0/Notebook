# 交易类型:

unlocking script + locking script

## Pay-to-Public-Key-Hash (P2PKH)
### locking script
```
OP_DUP OP_HASH160 <Cafe Public Key Hash> OP_EQUALVERIFY OP_CHECKSIG
```
### unlocking script
```
<Cafe Signature> <Cafe Public Key>
```

### 验证过程
![](img/2017-07-20-18-48-15.png)


## Multisignature
Multisignature scripts set a condition where N public keys are recorded in the script and at least M of those must provide signatures to unlock the funds
### locking script
```
M <Public Key 1> <Public Key 2> ... <Public Key N> N CHECKMULTISIG
```
### unlocking script
```
0 <Signature B> <Signature C>
```
## Pay-to-Script-Hash (P2SH)

