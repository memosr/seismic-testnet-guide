# Seismic Testnet: Şifreli Kontrat Deploy Rehberi

[![Network](https://img.shields.io/badge/network-Seismic%20Testnet-purple)](https://docs.seismic.systems)
[![Chain ID](https://img.shields.io/badge/chain%20id-5124-blue)](https://seismic-testnet.socialscan.io)
[![Language](https://img.shields.io/badge/lang-T%C3%BCrk%C3%A7e-red)](https://github.com/memosr/seismic-testnet-guide)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

Seismic, şifreli (encrypted) akıllı kontratlar için tasarlanmış EVM uyumlu bir Layer 1 blockchain. Bu rehber Seismic Testnet üzerinde kendi şifreli kontratını deploy etmeyi anlatıyor. Gizli oylama, gizli DeFi, fintech gibi use case'lerin temeli bu.

Rehberi macOS'ta baştan sona uygulayıp yaşadığım hataları da not aldım, o yüzden yer yer "burada takıldım" tipi uyarılar göreceksin.

## Ne yapacaksın

- macOS'ta Seismic dev araçlarını kuracaksın (`sforge`, `sanvil`, `scast`)
- Kalıcı bir dev wallet üreteceksin
- Faucet'ten SIZE (gas token) alacaksın
- Şifreli bir `Counter` kontratı deploy edeceksin
- 5 tane şifreli increment transaction atacaksın
- Threshold aşıldığında gizli değeri açıklayacaksın

Toplamda yaklaşık 10 dakika ve 6 on-chain transaction.

## Rehberin çıktısı

Bu adımları takip ederek 3 farklı şifreli kontrat deploy edildi ve 26+ transaction ile etkileşim doğrulandı.

| Kontrat | Adres | Tip | Explorer |
| --- | --- | --- | --- |
| Counter #1 | `0xBc6a061A02F46dA8E075b22461EA7699ECb3e87F` | Counter (threshold 5) | [Görüntüle](https://seismic-testnet.socialscan.io/address/0xBc6a061A02F46dA8E075b22461EA7699ECb3e87F) |
| Counter #2 | `0x3561cF5EB9e2307Ead367E71cdCDdE121D463DA1` | Counter (threshold 10) | [Görüntüle](https://seismic-testnet.socialscan.io/address/0x3561cF5EB9e2307Ead367E71cdCDdE121D463DA1) |
| PrivatePledgeTracker | `0xf125426b2b2C6d8B9aca9d9bE0a399a89Dd60886` | Shielded pledge accounting | [Görüntüle](https://seismic-testnet.socialscan.io/address/0xf125426b2b2C6d8B9aca9d9bE0a399a89Dd60886) |

Network: Seismic Testnet (Chain ID `5124`), RPC: `https://testnet-1.seismictest.net/rpc`

## Ön gereksinimler

- İşletim sistemi: macOS (M1/M2/M3) veya Linux. Windows'ta WSL2 gerekiyor.
- Homebrew: [brew.sh](https://brew.sh)
- GitHub hesabı: faucet en az 10 takipçi istiyor
- Disk alanı: ~500 MB
- Süre: ilk kurulum 30 dk, deploy 15 dk

## Adım 1: Sistem kontrolü

Mevcut araçları kontrol et:

```bash
echo "=== Homebrew ===" && brew --version 2>/dev/null || echo "YOK"
echo "=== Git ===" && git --version 2>/dev/null || echo "YOK"
echo "=== Rust ===" && rustc --version 2>/dev/null || echo "YOK"
echo "=== Bun ===" && bun --version 2>/dev/null || echo "YOK"
```

Eksik olanları Adım 2 ve 3'te kuracağız.

## Adım 2: Rust kurulumu

Rust, Seismic'in custom Foundry'sini compile etmek için gerekli.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

İnteraktif menü çıkacak, default kurulum için Enter'a bas.

Kurulum bitince:

```bash
source "$HOME/.cargo/env"
rustc --version && cargo --version
```

## Adım 3: Seismic Foundry (sfoundryup) kurulumu

Önce yükleyiciyi kur:

```bash
curl -L \
  -H "Accept: application/vnd.github.v3.raw" \
  "https://api.github.com/repos/SeismicSystems/seismic-foundry/contents/sfoundryup/install?ref=seismic" | bash
```

Script shell'ini (zsh/bash) otomatik tespit edip PATH'e ekliyor. PATH'i mevcut terminale yükle:

```bash
source ~/.zshenv
which sfoundryup
```

Sonra asıl araçları kur (`sforge`, `sanvil`, `scast`):

```bash
sfoundryup
```

Burada sudo şifresi sorabilir. `ssolc` (Seismic Solidity compiler) `/usr/local/bin/` dizinine kurulduğu için Mac kullanıcı şifreni istiyor.

## Adım 4: try-devnet repo'sunu klonla

Seismic'in resmi örnek kontratları bu repo'da:

```bash
mkdir -p ~/seismic-workspace && cd ~/seismic-workspace
git clone --recurse-submodules https://github.com/SeismicSystems/try-devnet.git
cd try-devnet
```

Repo'daki `config.sh` eski devnet URL'lerini kullanıyor, yeni testnet için güncellemek gerekiyor:

```bash
cat > config.sh << 'EOF'
#!/bin/bash

RPC_URL="https://testnet-1.seismictest.net/rpc"
FAUCET_URL="https://faucet.seismictest.net/"
EXPLORER_URL="https://seismic-testnet.socialscan.io"
EOF
```

## Adım 5: Kalıcı dev wallet üret

`try-devnet/packages/contract/script/deploy.sh` her çalıştığında yeni cüzdan üretiyor, dolayısıyla faucet'ten aldığın tokenler boşa gidiyor. Kalıcı bir cüzdan yapalım:

```bash
scast wallet new
```

Çıktı şuna benziyor:

```
Successfully created new keypair.
Address:     0x...
Private key: 0x...
```

Private key'ini kimseyle paylaşma ve ekrandan kapat. Güvenli bir yere kaydet:

```bash
mkdir -p ~/.seismic-wallet && chmod 700 ~/.seismic-wallet
nano ~/.seismic-wallet/dev.key
```

Nano açılınca private key'i yapıştır, sonra `Ctrl+O`, `Enter`, `Ctrl+X`. Ardından izinleri kilitle:

```bash
chmod 600 ~/.seismic-wallet/dev.key
```

## Adım 6: Faucet'ten SIZE al

Tarayıcıdan <https://faucet.seismictest.net/> adresine git:

1. GitHub ile login ol (10+ takipçi gerekiyor)
2. Cüzdan adresini yapıştır
3. "Request Tokens" butonuna bas
4. 10 SIZE düşecek

Her cüzdan 24 saatte bir kez talep edebiliyor.

Doğrulama:

```bash
scast balance <CÜZDAN_ADRESİN> --rpc-url https://testnet-1.seismictest.net/rpc
```

`10000000000000000000` görmelisin (10 × 10^18 wei = 10 SIZE).

Not: eski dokümanlarda geçen `gcp-1.seismictest.net` URL'i çalışmıyor, balance 0 dönüyor. Doğrusu `testnet-1.seismictest.net/rpc`.

## Adım 7: Deploy script'ini modifiye et

Script'i kalıcı cüzdanı kullanacak şekilde düzenleyelim. Önce yedek al:

```bash
cd ~/seismic-workspace/try-devnet/packages/contract
cp script/deploy.sh script/deploy.sh.backup
```

Sonra yeni script'i yaz:

```bash
cat > script/deploy.sh << 'EOF'
#!/bin/bash

set -e

source ../../config.sh
source ../common/print.sh

CONTRACT_PATH="src/Counter.sol:Counter"
DEPLOY_FILE="out/deploy.txt"
WALLET_FILE="$HOME/.seismic-wallet/dev.key"

prelude() {
    echo -e "${BLUE}Deploy an encrypted contract in <1m.${NC}"
    echo -e "It's a Counter contract that only reveals the counter once it's >=5."
    echo -ne "Press Enter to continue..."
    read -r
}

prelude

if [ ! -f "$WALLET_FILE" ]; then
    echo -e "${RED}Wallet key file not found at $WALLET_FILE${NC}"
    exit 1
fi

privkey=$(cat "$WALLET_FILE" | tr -d '[:space:]')
address=$(scast wallet address --private-key "$privkey")

echo -e "Using wallet: ${GREEN}$address${NC}"

balance=$(scast balance "$address" --rpc-url "$RPC_URL")
echo -e "Balance: ${GREEN}$balance${NC} wei"

if [ "$balance" == "0" ]; then
    echo -e "${RED}Balance is 0. Get funds from $FAUCET_URL${NC}"
    exit 1
fi

print_step "1" "Deploying contract"
deploy_output=$(sforge create \
    --rpc-url "$RPC_URL" \
    --private-key "$privkey" \
    --broadcast \
    "$CONTRACT_PATH" \
    --constructor-args 5)
print_success "Success."

print_step "2" "Summarizing deployment"
mkdir -p out
contract_address=$(echo "$deploy_output" | grep "Deployed to:" | awk '{print $3}')
tx_hash=$(echo "$deploy_output" | grep "Transaction hash:" | awk '{print $3}')
echo "$contract_address" >"$DEPLOY_FILE"
echo -e "Contract Address: ${GREEN}$contract_address${NC}"
echo -e "Transaction Hash: ${GREEN}$tx_hash${NC}"
echo -e "Contract Link: ${GREEN}$EXPLORER_URL/address/$contract_address${NC}"

echo -e "\n"
print_success "Success. You just deployed your first contract on Seismic!"
EOF
```

## Adım 8: Deploy

```bash
bash script/deploy.sh
```

"Press Enter" uyarısında Enter'a bas. 10-30 saniye içinde şöyle bir çıktı gelecek:

```
Using wallet: 0x...
Balance: 10000000000000000000 wei

Step 1: Deploying contract
Success.

Step 2: Summarizing deployment
Contract Address: 0x...
Transaction Hash: 0x...
Contract Link: https://seismic-testnet.socialscan.io/address/0x...
```

Kontrat canlı. Explorer linkine tıklayıp bakabilirsin.

## Adım 9: Counter kontratını anlamak

Deploy ettiğin `Counter.sol` şu yapıda:

```solidity
contract Counter {
  suint256 private number;    // şifreli sayaç, kimse göremez
  uint256 public threshold;   // public eşik, herkes görür

  function increment(suint256 amount) public { number += amount; }
  function getNumber() public view isThresholdReached returns (uint256) {
    return uint256(number);
  }

  modifier isThresholdReached() {
    require(number >= suint256(threshold), 'Threshold not reached');
    _;
  }
}
```

Anahtar nokta `suint256`. Bu Seismic'in shielded uint tipi. Standart `uint256`'ya `s` prefix'i eklendiğinde değer blockchain üzerinde şifreli depolanıyor. Ne `number`'ı ne de increment'lerin değerini kimse göremiyor.

## Adım 10: Şifreli etkileşim

Değişkenleri ayarla:

```bash
CONTRACT=<DEPLOY_ETTIGIN_ADRES>
PRIVKEY=$(cat ~/.seismic-wallet/dev.key | tr -d '[:space:]')
RPC=https://testnet-1.seismictest.net/rpc
```

İlk increment (gizli 1 ekle):

```bash
scast send --rpc-url $RPC --private-key $PRIVKEY $CONTRACT "increment(suint256)" 1
```

Burada `increment(uint256)` yazarsan revert alırsın. Fonksiyon imzası `increment(suint256)` olmalı.

4 increment daha, toplam 5:

```bash
for i in 1 2 3 4; do
  scast send --rpc-url $RPC --private-key $PRIVKEY $CONTRACT "increment(suint256)" 1
  sleep 2
done
```

## Adım 11: Şifreyi aç

Threshold (5) aşıldığı için `getNumber()` artık çalışmalı:

```bash
scast call --rpc-url $RPC $CONTRACT "getNumber()(uint256)"
```

Çıktı:

```
5
```

5 increment attın, blockchain hiçbirinin değerini bilmiyor ama toplam doğru çıkıyor. Seismic'in encrypted state mekanizması tam olarak bu.

## İleri seviye: kendi kontratını yaz

`Counter.sol` tutorial seviyesinde. Gerçek bir use case denemek istersen kendi şifreli kontratını yazabilirsin. Ben rehberi hazırlarken bunu da yaptım: bağış taahhütlerini şifreli tutan `PrivatePledgeTracker`.

### Tasarım yaklaşımı

Üç tür state var:

- Public: `address public owner` gibi, herkesin görmesi anlamlı bilgi
- Shielded: `suint256 private totalPledged` gibi, sadece kontratın yetkili fonksiyonları okuyabilir
- Hybrid: `hasPledged(address) → bool` gibi, "kim taahhüt etti" public ama "ne kadar" gizli

### Kontrat

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract PrivatePledgeTracker {
    address public owner;
    suint256 private totalPledged;
    mapping(address => suint256) private pledges;

    event Pledged(address indexed pledger);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function pledge(suint256 amount) public {
        require(uint256(amount) > 0, "Empty pledge");
        pledges[msg.sender] += amount;
        totalPledged += amount;
        emit Pledged(msg.sender);
    }

    function myPledge() public view returns (uint256) {
        return uint256(pledges[msg.sender]);
    }

    function getTotalPledged() public view onlyOwner returns (uint256) {
        return uint256(totalPledged);
    }

    function hasPledged(address pledger) public view returns (bool) {
        return uint256(pledges[pledger]) > 0;
    }
}
```

### Privacy modeli pratikte

Bu kontrata 4 bağış yaptım: 100, 50, 250, 75 (toplam 475). Sorgu sonuçları:

| Sorgu | Sonuç | Anlamı |
| --- | --- | --- |
| `hasPledged(senin_adres)` | `true` | Public, kim taahhüt etti görünüyor |
| `myPledge()` (off-chain caller) | `0` | `msg.sender = 0x0` olduğu için 0 dönüyor, gerçek miktar gizli |
| `eth_getStorageAt(slot 1)` | `0x0000...0000` | Storage layer'da bile şifreli, direkt okuyup 475 göremiyorsun |
| `scast balance kontrat` | gerçek SIZE balance | Public, gas bilgisi her zaman public |

Asıl önemli kısım burada: standart Ethereum'da `eth_getStorageAt(slot 1)` çağırsan 475 görürdün. Seismic'te `0x0` görüyorsun, yani `suint256` storage slot seviyesinde gerçekten şifreli.

### payable kullanma

İlk denememde `donate() payable` ve `suint256(msg.value)` kullandım. Sonuç:

- Compiler uyardı: `msg.value is always publicly visible on-chain. Assigning it to a shielded type does not hide the transaction value from observers.`
- Validator simülasyonda revert gördü, TX mempool'da takıldı
- Out-of-gas hatası verdi (515K gas yetmedi)

Doğru yaklaşım: `payable` yerine kullanıcıdan `suint256 amount` parametresi almak. Gerçek SIZE transferi gerekiyorsa bunu off-chain veya ayrı bir fonksiyonda yap. `msg.value` zaten tx metadata'sında public, onu shielded yapmaya çalışmak yanıltıcı oluyor.

### Deploy komutu

```bash
sforge create --rpc-url $RPC --private-key $PRIVKEY --broadcast \
  --gas-limit 3000000 \
  --gas-price 5000000000 \
  --priority-gas-price 2000000000 \
  "src/PrivatePledgeTracker.sol:PrivatePledgeTracker"
```

Gas ve priority değerlerini cömert tuttum, çünkü düşük değerlerle TX mempool'da takılıyor. Seismic testnet validator'ları minimum 1-2 Gwei priority bekliyor. "replacement transaction underpriced" hatası alırsan gas price'ı yükselt.

### Pledge et ve encrypted state'i oku

```bash
PLEDGE=<deploy_ettiğin_adres>

# Pledge at (amount=100, gizli)
scast send --rpc-url $RPC --private-key $PRIVKEY \
  --gas-limit 200000 --gas-price 5000000000 --priority-gas-price 2000000000 \
  $PLEDGE "pledge(suint256)" 100

# Public membership check
scast call --rpc-url $RPC $PLEDGE "hasPledged(address)(bool)" <senin_adresin>

# Storage slot: 0 dönmesi normal, şifreli olduğu anlamına geliyor
scast storage $PLEDGE 1 --rpc-url $RPC
```

## Sık karşılaşılan hatalar

Aşağıdaki hataların hepsini rehberi hazırlarken bizzat yaşadım.

| # | Hata | Sebep | Çözüm |
| --- | --- | --- | --- |
| 1 | `Address not funded` | Faucet henüz token göndermedi veya adres yanlış | Faucet'ten SIZE iste, 24h cooldown'a dikkat et |
| 2 | `scast balance 0` | `gcp-1.seismictest.net` artık aktif değil | RPC URL `https://testnet-1.seismictest.net/rpc` olmalı |
| 3 | `Failed to estimate gas: execution reverted` | Fonksiyon imzasında yanlış tip | `increment(uint256)` yerine `increment(suint256)` kullan |
| 4 | `command not found: sfoundryup` | PATH henüz yüklenmedi | `source ~/.zshenv` çalıştır veya yeni terminal aç |
| 5 | `libusb not found` uyarısı | Hardware wallet driver eksik | Devnet için gerekmiyor, yoksay |
| 6 | `Wallet key file not found at ~/.seismic-wallet/dev.key` | Cüzdan dosyası oluşturulmamış | `mkdir -p ~/.seismic-wallet && nano ~/.seismic-wallet/dev.key` |
| 7 | Cüzdan her deploy'da değişiyor | Default script `scast wallet new` üretiyor | Adım 7'deki kalıcı cüzdan modifikasyonunu uygula |
| 8 | `faucet-2.seismicdev.net` ulaşılmıyor | Eski devnet URL'i | Yeni faucet: `https://faucet.seismictest.net/` |
| 9 | `Error: encode length mismatch` (zsh) | bash array syntax'ı zsh'de farklı çalışıyor | Array yerine tek tek tx at veya `for i in 1 2 3` döngüsü kullan |
| 10 | `sforge build` Error 10109 | Compiler değişti, shielded type mapping key olarak kullanılamıyor | Repo'da bug var, [issue #10](https://github.com/SeismicSystems/prototypes/issues/10) açıldı |
| 11 | `donate()` payable + `suint256(msg.value)` çalışmıyor | `msg.value` zaten public, shielded'a cast'lemek anlamsız | `payable` yerine `pledge(suint256 amount)` gibi parametre al |

## Faydalı linkler

- Seismic: <https://www.seismic.systems>
- Dokümantasyon: <https://docs.seismic.systems>
- GitHub: <https://github.com/SeismicSystems>
- Discord: <https://discord.gg/XSPNseXCvW>
- X: <https://x.com/SeismicSys>
- Faucet: <https://faucet.seismictest.net>
- Explorer: <https://seismic-testnet.socialscan.io>

## Lisans

MIT. Fork'la, kendine göre düzenle.
