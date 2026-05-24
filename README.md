#  Seismic Testnet — Encrypted Smart Contract Deploy Rehberi

[![Network](https://img.shields.io/badge/network-Seismic%20Testnet-purple)](https://docs.seismic.systems)
[![Chain ID](https://img.shields.io/badge/chain%20id-5124-blue)](https://seismic-testnet.socialscan.io)
[![Language](https://img.shields.io/badge/lang-Türkçe-red)](https://github.com/memosr/seismic-testnet-guide)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

Seismic, **şifreli (encrypted) akıllı kontratlar** için tasarlanmış, EVM uyumlu bir Layer 1 blockchain'dir. Bu rehber, Seismic Testnet üzerinde **kendi şifreli kontratını deploy etmeyi** adım adım anlatır — fintech, gizli oylama, gizli DeFi gibi use case'lerin temeli.

##  Bu Rehberde Ne Yapacaksın?

- ✅ macOS'ta Seismic dev araçlarını (sforge, sanvil, scast) kuracaksın
- ✅ Yeni bir dev wallet üreteceksin
- ✅ Faucet'ten SIZE (gas token) alacaksın
- ✅ Şifreli bir `Counter` kontratı deploy edeceksin
- ✅ 5 adet **şifreli increment transaction** atacaksın
- ✅ Threshold aşıldığında **gizli değeri açıklayacaksın**

**Sonuç:** ~10 dakikada 6 on-chain transaction, encrypted smart contract deneyimi.

##  Kanıt — Bu Rehberin Çıktısı

Bu rehberi takip ederek **3 farklı şifreli kontrat** deploy edildi ve **26+ on-chain transaction** ile etkileşim doğrulandı:

| Kontrat | Adres | Tip | Explorer |
|---|---|---|---|
| **Counter #1** | `0xBc6a061A02F46dA8E075b22461EA7699ECb3e87F` | Counter (threshold 5) | [Görüntüle](https://seismic-testnet.socialscan.io/address/0xBc6a061A02F46dA8E075b22461EA7699ECb3e87F) |
| **Counter #2** | `0x3561cF5EB9e2307Ead367E71cdCDdE121D463DA1` | Counter (threshold 10) | [Görüntüle](https://seismic-testnet.socialscan.io/address/0x3561cF5EB9e2307Ead367E71cdCDdE121D463DA1) |
| **PrivatePledgeTracker** | `0xf125426b2b2C6d8B9aca9d9bE0a399a89Dd60886` | Özgün — shielded pledge accounting | [Görüntüle](https://seismic-testnet.socialscan.io/address/0xf125426b2b2C6d8B9aca9d9bE0a399a89Dd60886) |

**Network:** Seismic Testnet (Chain ID: `5124`) · **RPC:** `https://testnet-1.seismictest.net/rpc`

## 📋 Ön Gereksinimler

- **İşletim Sistemi:** macOS (M1/M2/M3) veya Linux. Windows için WSL2 gerekir.
- **Homebrew:** macOS paket yöneticisi → [brew.sh](https://brew.sh)
- **GitHub hesabı:** Minimum **10 takipçi** (faucet için zorunlu)
- **Disk alanı:** ~500 MB
- **Süre:** İlk kurulum 30 dk, deploy 15 dk

##  Adım 1 — Sistem Kontrolü

Terminalde mevcut araçları kontrol et:

```bash
echo "=== Homebrew ===" && brew --version 2>/dev/null || echo "❌ YOK"
echo "=== Git ===" && git --version 2>/dev/null || echo "❌ YOK"
echo "=== Rust ===" && rustc --version 2>/dev/null || echo "❌ YOK"
echo "=== Bun ===" && bun --version 2>/dev/null || echo "❌ YOK"
```

Eksik olanlar varsa Adım 2-3'te kuracağız.

##  Adım 2 — Rust Kurulumu

Rust, Seismic'in custom Foundry'sini compile etmek için gerekli.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

İnteraktif menü çıkacak. **Enter** tuşuna bas (default kurulum).

Kurulum bitince:

```bash
source "$HOME/.cargo/env"
rustc --version && cargo --version
```

##  Adım 3 — Seismic Foundry (sfoundryup) Kurulumu

Seismic'in özel Foundry araç setini yükleyen yükleyici.

```bash
curl -L \
  -H "Accept: application/vnd.github.v3.raw" \
  "https://api.github.com/repos/SeismicSystems/seismic-foundry/contents/sfoundryup/install?ref=seismic" | bash
```

Script otomatik olarak shell'ini (zsh/bash) tespit edip PATH'e ekler.

PATH'i mevcut terminale yükle:

```bash
source ~/.zshenv
which sfoundryup
```

Sonra asıl araçları kur (`sforge`, `sanvil`, `scast`):

```bash
sfoundryup
```

> ⚠️ **Sudo şifresi sorabilir** — `ssolc` (Seismic Solidity compiler) `/usr/local/bin/` dizinine kurulduğu için Mac kullanıcı şifreni isteyecek.

##  Adım 4 — try-devnet Repo'sunu Klonla

Seismic'in resmi örnek kontratlarını içeren repo:

```bash
mkdir -p ~/seismic-workspace && cd ~/seismic-workspace
git clone --recurse-submodules https://github.com/SeismicSystems/try-devnet.git
cd try-devnet
```

>  **Önemli — Config Güncelleme Gerek:** Bu repo'daki `config.sh` dosyası **eski devnet URL'lerini** kullanıyor. Yeni testnet için güncelleyelim.

```bash
cat > config.sh << 'EOF'
#!/bin/bash

RPC_URL="https://testnet-1.seismictest.net/rpc"
FAUCET_URL="https://faucet.seismictest.net/"
EXPLORER_URL="https://seismic-testnet.socialscan.io"
EOF
```

##  Adım 5 — Kalıcı Dev Wallet Üret

`try-devnet/packages/contract/script/deploy.sh` her seferinde **yeni** cüzdan üretiyor. Bu yüzden faucet'ten alınan tokenler kaybolur. Kalıcı bir cüzdan yapacağız.

```bash
scast wallet new
```

Çıktı şuna benzer:

```
Successfully created new keypair.
Address:     0x... 
Private key: 0x...
```

⚠️ **GÜVENLİK:** Private key'ini **kimseyle paylaşma**, ekrandan da kapat. Şimdi güvenli bir yere kaydet:

```bash
mkdir -p ~/.seismic-wallet && chmod 700 ~/.seismic-wallet
nano ~/.seismic-wallet/dev.key
```

Nano editörü açılınca **private key'i yapıştır**, sonra `Ctrl+O` → `Enter` → `Ctrl+X`.

İzinleri kilitle:

```bash
chmod 600 ~/.seismic-wallet/dev.key
```

##  Adım 6 — Faucet'ten SIZE Al

Tarayıcıdan: **https://faucet.seismictest.net/**

1. **GitHub ile login** (>10 takipçi gerekli)
2. Cüzdan adresini yapıştır
3. "Request Tokens" butonuna bas
4. **10 SIZE** düşecek

>  **Cooldown:** Her cüzdan için **24 saatte 1 kez** talep edebilirsin.

Doğrulama:

```bash
scast balance <CÜZDAN_ADRESİN> --rpc-url https://testnet-1.seismictest.net/rpc
```

`10000000000000000000` (10 × 10^18 wei = 10 SIZE) görmelisin.

>  **Yaygın Hata — RPC URL:** Eski docs'ta `gcp-1.seismictest.net` URL'i geçiyor, **çalışmıyor** (balance 0 dönüyor). Doğrusu: `testnet-1.seismictest.net/rpc`

##  Adım 7 — Deploy Script'ini Modifiye Et

`try-devnet/packages/contract/script/deploy.sh` script'i her seferinde yeni cüzdan üretiyor. Bizim kalıcı cüzdanı kullanacak şekilde düzenleyelim.

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

##  Adım 8 — Deploy!

```bash
bash script/deploy.sh
```

**Press Enter** uyarısında Enter'a bas. ~10-30 saniye içinde:

```
Using wallet: 0x...
Balance: 10000000000000000000 wei

Step 1: Deploying contract
✅ Success.

Step 2: Summarizing deployment
Contract Address: 0x...
Transaction Hash: 0x...
Contract Link: https://seismic-testnet.socialscan.io/address/0x...
```

 Kontratın canlı! Explorer linkine tıkla, ziyaret et.

##  Adım 9 — Counter Kontratı Anlamak

Deploy ettiğin `Counter.sol` kontratı şu yapıda:

```solidity
contract Counter {
  suint256 private number;    // 🔒 ŞİFRELİ sayaç (kimse göremez)
  uint256 public threshold;   // 🌐 PUBLIC eşik (herkes görür)

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

**Anahtar nokta:** `suint256` Seismic'in **shielded uint** tipidir. Standart `uint256`'ya `s` prefix'i eklenince blockchain üzerinde **şifreli** depolanır. Kimse ne `number`'ı görebilir ne de `increment`'lerin değerini.

##  Adım 10 — Şifreli Etkileşim

Değişkenleri ayarla:

```bash
CONTRACT=<DEPLOY_ETTIGIN_ADRES>
PRIVKEY=$(cat ~/.seismic-wallet/dev.key | tr -d '[:space:]')
RPC=https://testnet-1.seismictest.net/rpc
```

İlk increment'i çağır (gizli 1 ekle):

```bash
scast send --rpc-url $RPC --private-key $PRIVKEY $CONTRACT "increment(suint256)" 1
```

>  **Yaygın Hata:** `increment(uint256)` yazarsan **revert** alırsın. Doğrusu **`increment(suint256)`** — shielded type.

4 increment daha (toplam 5):

```bash
for i in 1 2 3 4; do
  scast send --rpc-url $RPC --private-key $PRIVKEY $CONTRACT "increment(suint256)" 1
  sleep 2
done
```

##  Adım 11 — Şifreyi Aç!

Threshold (5) aşıldı, `getNumber()` artık çalışmalı:

```bash
scast call --rpc-url $RPC $CONTRACT "getNumber()(uint256)"
```

Çıktı:

```
5
```

 **İşte bu!** 5 increment yaptın, blockchain hiçbirinin değerini bilmiyor, ama toplam **doğru** çıktı. Bu Seismic'in encrypted state mekanizmasının kanıtı.

## 🚀 İleri Seviye — Kendi Özgün Kontratını Yaz

`Counter.sol` tutorial seviyesinde. Gerçek bir use case yaratmak istiyorsan kendi şifreli kontratını yazabilirsin. Bu rehberi yazarken bunu da yaptık: **PrivatePledgeTracker** adında bağış taahhütlerini şifreli tutan bir kontrat.

### Tasarım Felsefesi

Üç tür state var:
- 🌐 **Public** → `address public owner` gibi, herkesin görmesi anlamlı bilgi
- 🔒 **Shielded** → `suint256 private totalPledged` gibi, sadece kontratın yetkili fonksiyonları okuyabilir
- 🟡 **Hybrid** → `hasPledged(address) → bool` — "kim taahhüt etti" public ama "ne kadar" gizli

### Tam Kontrat Örneği

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

### Privacy Model — Net Gösterim

Bu kontratta 4 farklı bağış yaptık: **100, 50, 250, 75** (toplam 475). Sonuçlar:

| Sorgu | Sonuç | Anlamı |
|---|---|---|
| `hasPledged(senin_adres)` | `true` | 🌐 Public — kim taahhüt etti görünür |
| `myPledge()` (off-chain caller) | `0` | 🔒 **`msg.sender = 0x0` olduğu için 0 görünür** — gerçek miktar gizli |
| `eth_getStorageAt(slot 1)` | `0x0000...0000` | 🔒 **Storage layer'da bile şifreli!** Direkt okuyup 475 göremezsin |
| `scast balance kontrat` | gerçek SIZE balance | 🌐 Public — gas bilgisi her zaman public |

> **🎯 Büyük zafer:** Standart Ethereum'da `eth_getStorageAt(slot 1)` çağırsan **475** görürdün. Seismic'te `0x0` görüyorsun — `suint256` storage slot seviyesinde gerçekten şifreli.

### Önemli Tasarım Notu — `payable` Kullanma!

İlk denemede `donate() payable` ve `suint256(msg.value)` kullanmıştık. Sonuç:

- ⚠️ Compiler uyardı: `msg.value is always publicly visible on-chain. Assigning it to a shielded type does not hide the transaction value from observers.`
- ❌ Validator simülasyonda revert gördü, TX mempool'da takıldı
- ❌ Out-of-gas hatası verdi (515K gas yetmedi)

**Doğru yaklaşım:** `payable` yerine kullanıcıdan **`suint256 amount` parametresi** al. Gerçek SIZE transferi gerekiyorsa bunu off-chain veya ayrı bir fonksiyonda yap. Çünkü `msg.value` zaten tx metadata'sında public — onu shielded yapmaya çalışmak yanıltıcıdır.

### Deploy Komutu (Cömert Gas ile)

```bash
sforge create --rpc-url $RPC --private-key $PRIVKEY --broadcast \
  --gas-limit 3000000 \
  --gas-price 5000000000 \
  --priority-gas-price 2000000000 \
  "src/PrivatePledgeTracker.sol:PrivatePledgeTracker"
```

> ⚠️ **Düşük gas/priority ile TX mempool'da takılır.** Seismic testnet validator'ları minimum 1-2 Gwei priority bekliyor. Eğer "replacement transaction underpriced" hatası alırsan gas price'ı yükselt.

### Pledge Et + Encrypted State'i Oku

```bash
PLEDGE=<deploy_ettiğin_adres>

# Pledge at (amount=100, gizli)
scast send --rpc-url $RPC --private-key $PRIVKEY \
  --gas-limit 200000 --gas-price 5000000000 --priority-gas-price 2000000000 \
  $PLEDGE "pledge(suint256)" 100

# Public membership check
scast call --rpc-url $RPC $PLEDGE "hasPledged(address)(bool)" <senin_adresin>

# Storage slot (encrypted, 0 görmeli — bu doğru!)
scast storage $PLEDGE 1 --rpc-url $RPC
```

##  Sık Karşılaşılan Hatalar

Aşağıdaki tüm hatalar bu rehberi hazırlarken **gerçek olarak yaşandı** ve burada çözümleriyle birlikte belgelendi:

| # | Hata | Sebep | Çözüm |
|---|---|---|---|
| 1 | `Address not funded` | Faucet henüz token göndermedi veya yanlış adres | Faucet'ten SIZE iste, 24h cooldown'a dikkat |
| 2 | `scast balance 0` (eski RPC) | `gcp-1.seismictest.net` artık aktif değil | RPC URL'i `https://testnet-1.seismictest.net/rpc` olmalı |
| 3 | `Failed to estimate gas: execution reverted` | Fonksiyon imzasında yanlış tip | `increment(uint256)` yerine `increment(suint256)` kullan |
| 4 | `command not found: sfoundryup` | PATH henüz yüklenmedi | `source ~/.zshenv` çalıştır veya yeni terminal aç |
| 5 | `libusb not found` warning | Hardware wallet driver eksik | Devnet için gerek yok, yoksay |
| 6 | `Wallet key file not found at ~/.seismic-wallet/dev.key` | Cüzdan dosyası oluşturulmamış | `mkdir -p ~/.seismic-wallet && nano ~/.seismic-wallet/dev.key` |
| 7 | Cüzdan her deploy'da değişiyor | Default script `scast wallet new` üretiyor | Adım 7'deki kalıcı cüzdan modifikasyonunu uygula |
| 8 | `faucet-2.seismicdev.net` ulaşılmıyor | Eski devnet URL'i | Yeni faucet: `https://faucet.seismictest.net/` |
| 9 | `Error: encode length mismatch` (zsh) | bash array syntax'ı zsh'de farklı | bash array kullanmak yerine tek tek tx at veya açık `for i in 1 2 3` döngüsü kullan |
| 10 | `sforge build` Error 10109 | Compiler değişti: shielded type mapping key olarak kullanılamaz | Bu repo'da bug; [issue #10](https://github.com/SeismicSystems/prototypes/issues/10) açıldı |
| 11 | `donate()` payable + `suint256(msg.value)` çalışmıyor | Compiler uyarısı: `msg.value` zaten public, shielded'a cast'lemek anlamsız | `payable` yerine `pledge(suint256 amount)` gibi parametre al, gerçek transfer off-chain veya ayrı fonksiyonda |

## 🔗 Faydalı Linkler

- 🌐 **Seismic Resmi:** https://www.seismic.systems
- 🌐 **Dökümantasyon:** https://docs.seismic.systems
- 🌐 **GitHub:** https://github.com/SeismicSystems
- 🌐 **Discord:** https://discord.gg/XSPNseXCvW
- 🌐 **X (Twitter):** https://x.com/SeismicSys
- 🌐 **Faucet:** https://faucet.seismictest.net
- 🌐 **Explorer:** https://seismic-testnet.socialscan.io

## 📜 Lisans

MIT — kullan, fork'la, kendi rehberini yap.

---

**Bu rehber topluluğa katkı amacıyla hazırlanmıştır.** Bir hata bulursan ya da iyileştirme önerin varsa **issue** veya **pull request** açabilirsin.
