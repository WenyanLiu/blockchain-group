# Sender(singer Singer, tx *Transaction)

在交易的结构体中，交易接收者的地址被显式地声明，交易发起者使用数字签名`V`、`R`和`S`表示。

```go
// core/types/transaction.go
type Transaction struct {
	data txdata
	// caches
	hash atomic.Value
	size atomic.Value
	from atomic.Value
}

type txdata struct {
	AccountNonce uint64          `json:"nonce"    gencodec:"required"`
	Price        *big.Int        `json:"gasPrice" gencodec:"required"`
	GasLimit     uint64          `json:"gas"      gencodec:"required"`
	Recipient    *common.Address `json:"to"       rlp:"nil"` // nil means contract creation
	Amount       *big.Int        `json:"value"    gencodec:"required"`
	Payload      []byte          `json:"input"    gencodec:"required"`

	// Signature values
	V *big.Int `json:"v" gencodec:"required"`
	R *big.Int `json:"r" gencodec:"required"`
	S *big.Int `json:"s" gencodec:"required"`

	// This is only used when marshaling to JSON.
	Hash *common.Hash `json:"hash" rlp:"-"`
}
```

当[ApplyTransaction()](./ApplyTransaction.md#%E4%B8%80-message)恢复交易的发起者地址时，先从交易的签名中恢复出公钥，再将公钥转化为`Address`类型的地址。对此，定义`Signer`接口：

```go
// core/types/transaction_signing.go
type Signer interface {
	Sender(tx *Transaction) (common.Address, error)
	SignatureValues(tx *Transaction, sig []byte) (r, s, v *big.Int, err error)
	Hash(tx *Transaction) common.Hash
	Equal(Signer) bool
}
```

`SignTx`函数生成数字签名：

```go
// core/types/transaction_signing.go
func SignTx(tx *Transaction, s Signer, prv *ecdsa.PrivateKey) (*Transaction, error) {
	log.DebugLog()
	h := s.Hash(tx)
	sig, err := crypto.Sign(h[:], prv)
	if err != nil {
		return nil, err
	}
	return tx.WithSignature(s, sig)
}
```

`Sender`函数恢复交易发起者的地址。其中，`signer.Sender`从数字签名的字符串中恢复出公钥，并转化为地址：

```go
// core/types/transaction_signing.go
func Sender(signer Signer, tx *Transaction) (common.Address, error) {
	log.DebugLog()
	if sc := tx.from.Load(); sc != nil {
		sigCache := sc.(sigCache)		if sigCache.signer.Equal(signer) {
			return sigCache.from, nil
		}
	}

	addr, err := signer.Sender(tx)
	if err != nil {
		return common.Address{}, err
	}
	tx.from.Store(sigCache{signer: signer, from: addr})
	return addr, nil
}
```

至此，解析出交易对象的交易发起者的地址，`Transaction`被封装成`Message`类型。

