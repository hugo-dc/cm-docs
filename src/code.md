# Code

## Stats (ContractBag Method)

```go
func (b *ContractBag) Stats() (*CMStats, error) {
	stats := &CMStats{
		NumContracts:  len(b.contracts),
		ProofStats:    &ProofStats{},
		TouchedChunks: make([]int, 0, len(b.contracts)),
	}
	for _, v := range b.contracts {
		stats.CodeSize += v.CodeSize()
		ps, err := v.ProofStats()
		if err != nil {
			return nil, err
		}
		stats.ProofStats.Add(ps)
		stats.TouchedChunks = append(stats.TouchedChunks, ps.TouchedChunks)
	}
	stats.ProofSize = stats.ProofStats.Sum()
	return stats, nil
}
```


