## pcpCrypto

pcpCrypto implements the Go standard [crypto.Signer](https://golang.org/pkg/crypto/#Signer) interface for keys that live inside Microsoft's TPM Platform Crypto Provider (PCP). All operations are performed by calling PCP's [Key Storage Provider (KSP)](https://docs.microsoft.com/en-us/windows/win32/seccertenroll/cng-key-storage-providers), using [CNG Key Storage Functions
](https://docs.microsoft.com/en-us/windows/win32/seccng/cng-key-storage-functions)

This allows the use of TPM PCP keys in operations which require `crypto.Signer` keys (i.e. [x509.CreateCertificateRequest](https://golang.org/pkg/crypto/x509/#CreateCertificateRequest))

For the moment, pcpCrypto only implements the following (more is yet to come) :

* Generation and retrieval of RSA and ECDSA PCP keys.
* RSA digest signing using PKCS#1 v1.5 & PSS schemes.
* ECDSA digest signing.

## Installation

The pcpCrypto package is installable using go get: `go get github.com/ElMostafaIdrassi/pcpcrypto`.

## General trivia about PCP TPM keys

Each PCP-generated TPM key is persistent in regards to the PCP KSP : it has a name and persists
through reboots. 

In order to achieve this persistence, the PCP KSP creates, for each PCP-generated TPM key, a corresponding file which contains all the information about the public and private parts of the key. 
For a key that applies to the current user `<username>`, this file resides in `C:\Users\<username>\AppData\Local\Microsoft\Crypto\PCPKSP\<SHA1DIGEST\`, has a name that is the `SHA-1 digest` of the key's name and the extension `PCPKEY`. 
For a key that applies to the local machine, this file resides in `C:\ProgramData\Microsoft\Crypto\PCPKSP\<SHA1DIGEST\`, has a name that is the 
`SHA-1 digest` of the key's name and the extension `PCPKEY`.

This gives the PCP KSP the ability to actually load the key into the TPM's volatile memory at each `NCryptOpenKey` and unload it at each `NCryptFreeObject`, in the manner of a `TPM2-TSS-ENGINE` encrypted blob file.

Therefore, PCP KSP keys are not persistent in regards to the TPM chip itself. They are merely transient keys that are created inside of the TPM using `NCryptCreatePersistedKey` and exported immediately into the aforementioned PCP file when `NCryptFinalizeKey` is called. These PCP files will then allow the PCP KSP to reload the key into TPM's volatile memory whenever needed.

## Known Limitations

* The PCP KSP fails to sign using `PKCS#1 PSS` scheme, whatever the passed salt length is.
The reason is that TPM chips come in 2 variations : 
  * Chips that follow the Pre-TPM Spec-1.16 : these chips use a salt length that is always equal to the maximum allowed salt length, which is given by:
``
keySizeInBytes - digestSizeInBytes - 2
``
  * Chips that follow the Post-TPM Spec-1.16 : these chips use a salt length that is always equal to the hash length.

  This means that the PCP KSP needs to use the salt length that the TPM chip supports instead of the one chosen by the caller. Therefore, the PCP KSP developers have given us the flag `NCRYPT_TPM_PAD_PSS_IGNORE_SALT` which needs to be passed during the signature. This option forces the PCP KSP to ignore the passed salt length and to always use the one that is supported by the TPM chip. This makes some TPMs incompatible with implementations that require the salt length to be equal to the hash length (i.e. TLS 1.3).

* Even with the `NCRYPT_TPM_PAD_PSS_IGNORE_SALT` hack, PCP KSP can only sign `SHA1` and `SHA256` digests and fails to sign `SHA384` and `SHA512` digests using `PKCS#1 PSS` scheme with error `NTE_NOT_SUPPORTED`.

* The PCP KSP fails to sign `SHA384` and `SHA512` digests using an `ECDSA NIST P256` key with error code `0x802801D5`. This means the KSP does not truncate the digests that are longer than the curve's bit size before signing.