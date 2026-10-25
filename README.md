# LA TANDA

<img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/46ef6f9d-ffc4-46e5-8e32-5a91b9401c66" />

# La Tanda Testnet Validatör Rehberi (Cosmovisor ile)


**La Tanda Chain** testnet'inde (`latanda-testnet-1`, Cosmos SDK v0.53.6 / CometBFT) **Cosmovisor** ile validatör çalıştırmak için sahada denenmiş pratik bir rehber. Discord doğrulama adımlarını (`!verify` ve HTTP-nonce altyapı kontrolü) ve yol boyunca yapılan hataları da içerir.

> **Uyarı.** Bu rehber resmi değil, topluluk rehberidir. Herkese açık UTSA topluluk rehberini ve La Tanda ekibinin forum yazılarını baz alır, gerçek kullanımda çıkan sorunların çözümlerini ekler. Projeye göre testnet LTD'nin parasal değeri yoktur. Programlar, endpoint'ler ve kurallar değişebilir, bu yüzden projenin Discord'undaki sabitlenmiş mesajları mutlaka kontrol et. Son test: Ekim 2026.

## İçindekiler

1. [Hızlı bilgiler](#1-hızlı-bilgiler)
2. [Gereksinimler](#2-gereksinimler)
3. [Değişkenler](#3-değişkenler)
4. [Ön kontroller](#4-ön-kontroller)
5. [Binary'yi derle](#5-binaryyi-derle)
6. [Cosmovisor dizin yapısı](#6-cosmovisor-dizin-yapısı)
7. [Init ve genesis](#7-init-ve-genesis)
8. [Konfigürasyon](#8-konfigürasyon)
9. [State sync](#9-state-sync)
10. [systemd servisi](#10-systemd-servisi)
11. [Cüzdan ve faucet](#11-cüzdan-ve-faucet)
12. [Validatörü oluştur](#12-validatörü-oluştur)
13. [Discord doğrulaması](#13-discord-doğrulaması)
14. [İzleme](#14-izleme)
15. [Cosmovisor ile güncelleme](#15-cosmovisor-ile-güncelleme)
16. [Yedekleme ve güvenlik](#16-yedekleme-ve-güvenlik)
17. [Sorun giderme](#17-sorun-giderme)
18. [Kaynaklar](#18-kaynaklar)

## 1. Hızlı bilgiler

| Bilgi | Değer |
| --- | --- |
| Chain ID | `latanda-testnet-1` |
| Binary / CLI | `latandad` |
| Home dizini | `~/.latanda` |
| Denom | `ultd` (**1 LTD = 1.000.000 ultd**, 6 ondalık) |
| Adres öneki | `ltd` (validatör operatörü: `ltdvaloper`, konsensüs: `ltdvalcons`) |
| Blok süresi | yaklaşık 5 saniye |
| Minimum gas fiyatı | `0.001ultd` (tipik ücret `5000ultd`) |
| Faucet | Discord, kişi başı 10 LTD, bir kez (bkz. adım 11) |
| Validatör stake'i | protokol seviyesinde minimum yok, aktif set en fazla 100 validatör |
| Genesis | `https://latanda.online/chain/genesis.json` |
| Genesis SHA-256 | `98fc9871d6a3b7b12b3f7fcaa1ca3303ffcfad0f209d61355975a15069ac3907` |

**Birim uyarısı.** Upstream rehberdeki örnekte tutar `1100000000000000000ultd` yazıyor. Bu 18 ondalık varsayar ama zincir 6 ondalık kullanıyor. 10 LTD'lik faucet bakiyesiyle (10.000.000 ultd) bu işlem `insufficient funds` hatasıyla başarısız olur. Bunun yerine `1000000ultd` (1 LTD) kullan. Ayrıntısı adım 12'de.

## 2. Gereksinimler

- Ubuntu 22.04 ya da üstü, 2 vCPU, 4 GB RAM, 100 GB SSD testnet için yeterli
- Root ya da sudo yetkili kullanıcı
- Go 1.23 ya da üstü (derleme daha yeni bir toolchain'i otomatik indirebilir, bkz. adım 5)
- `curl wget jq git make gcc tar lz4`
- `cosmovisor` (eksikse adım 4'te kurulur)
- Sunucunun bot koruması captcha'sını geçebilen bir Discord hesabı
- P2P için boş bir TCP portu (örneklerde `41xxx` aralığı kullanılıyor, gerekirse değiştir)

## 3. Değişkenler

Bunları her terminal oturumunda bir kez tanımla. Bağlantı kopup yeniden bağlanırsan tekrar tanımla.

```bash
export MONIKER="<validator-adin>"
export WALLET="<anahtar-adin>"
export P=41                         # port öneki, 41656, 41657 gibi portlar verir
export CHAIN_ID=latanda-testnet-1
export CFG=$HOME/.latanda/config
```

`P` öneki, makinede başka Cosmos node'ları çalışıyorsa önemli. Varsayılan 26656 ve 26657 portları genelde doludur. Makinede sadece bu node çalışacaksa varsayılan portları koruyabilir ve adım 8'deki port düzenlemelerini atlayabilirsin.

## 4. Ön kontroller

```bash
# çıktı boş olmalı, doluysa başka bir önek seç
ss -tlnp | grep -E ":${P}[0-9]{3} "

go version
which cosmovisor || go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0

for t in curl wget jq git make gcc tar lz4; do command -v $t >/dev/null || echo "eksik: $t"; done
```

Bir araç eksikse sadece onu `apt install <paket>` ile kur.

## 5. Binary'yi derle

```bash
mkdir -p $HOME/latanda-build && cd $HOME/latanda-build
wget -q https://latanda.online/chain/latanda-chain-source.tar.gz
tar -xzf latanda-chain-source.tar.gz
grep -E '^(go|toolchain) ' go.mod
go mod tidy
go build -o latandad ./cmd/latandad
sha256sum latanda-chain-source.tar.gz
```

Notlar:

- Zincir, projenin kendi kaynak tarball'ından derleniyor. Karşılaştırabileceğin bir release checksum'ı yok. Tarball'ın SHA-256 değerini Discord'da diğer operatörlerle karşılaştır, ama bunun sadece aynı dosyayı indirdiğini gösterdiğini, kodun güvenli olduğunu göstermediğini unutma.
- `go.mod` kurulu Go'dan daha yeni bir sürüm istiyorsa ve Go'n 1.21 ya da üstüyse, toolchain sadece bu derleme için otomatik indirilir. Sistemdeki Go'ya dokunulmaz. (Bizim denememizde Go 1.25.6 bu şekilde indi.)
- `latandad version --long` komutu `version` ve `commit` alanlarını boş gösterir. Bu derleme için beklenen bir durum.

## 6. Cosmovisor dizin yapısı

```bash
mkdir -p $HOME/.latanda/cosmovisor/genesis/bin $HOME/.latanda/cosmovisor/upgrades
cp $HOME/latanda-build/latandad $HOME/.latanda/cosmovisor/genesis/bin/
ln -sfn $HOME/.latanda/cosmovisor/genesis $HOME/.latanda/cosmovisor/current
ln -sfn $HOME/.latanda/cosmovisor/current/bin/latandad /usr/local/bin/latandad
latandad version
```

Son symlink sayesinde `latandad` komutu Cosmovisor'un o an kullandığı binary'yi takip eder. Böylece güncellemeden sonra CLI her zaman çalışan node ile aynı sürümde olur.

## 7. Init ve genesis

```bash
latandad init "$MONIKER" --chain-id $CHAIN_ID --default-denom ultd

wget -O $CFG/genesis.json "https://latanda.online/chain/genesis.json"
sha256sum $CFG/genesis.json
# beklenen: 98fc9871d6a3b7b12b3f7fcaa1ca3303ffcfad0f209d61355975a15069ac3907

wget -O $CFG/addrbook.json "https://share101.utsa.tech/latanda/addrbook.json"
```

**Genesis hash'i tutmuyorsa dur.** Devam etmeden önce Discord'dan güncel hash'i sor.

## 8. Konfigürasyon

### client.toml

```bash
sed -i -e "s|^chain-id *=.*|chain-id = \"$CHAIN_ID\"|" \
       -e "s|^keyring-backend *=.*|keyring-backend = \"file\"|" \
       -e "s|^node *=.*|node = \"tcp://localhost:${P}657\"|" $CFG/client.toml
```

`node` satırı önemli. Varsayılan değer 26657 portunu gösterir. Birden fazla node çalışan bir makinede bu büyük ihtimalle başka bir zincirin RPC'sidir ve CLI komutların fark etmeden ona gider.

### app.toml

```bash
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.001ultd\"|" \
       -e "s|^pruning *=.*|pruning = \"custom\"|" \
       -e "s|^pruning-keep-recent *=.*|pruning-keep-recent = \"1000\"|" \
       -e "s|^pruning-interval *=.*|pruning-interval = \"10\"|" \
       -e "s|^address = \"tcp://localhost:1317\"|address = \"tcp://localhost:${P}317\"|" \
       -e "s|^address = \"localhost:9090\"|address = \"localhost:${P}090\"|" $CFG/app.toml
```

### config.toml

```bash
sed -i -e "s|^proxy_app = .*|proxy_app = \"tcp://127.0.0.1:${P}658\"|" \
       -e "s|^laddr = \"tcp://127.0.0.1:26657\"|laddr = \"tcp://127.0.0.1:${P}657\"|" \
       -e "s|^laddr = \"tcp://0.0.0.0:26656\"|laddr = \"tcp://0.0.0.0:${P}656\"|" \
       -e "s|^pprof_laddr = .*|pprof_laddr = \"localhost:${P}060\"|" \
       -e "s|^prometheus_listen_addr = .*|prometheus_listen_addr = \":${P}660\"|" \
       -e "s|^external_address *=.*|external_address = \"$(curl -4 -s https://api.ipify.org):${P}656\"|" \
       -e "s|^persistent_peers *=.*|persistent_peers = \"483a8110c3cd93c8dd3801d935151e98656f5b67@168.231.67.201:26656,d6217d85d3747ebbfbf75898bd4407da567f5291@65.108.206.118:60556\"|" $CFG/config.toml
```

Upstream rehber `persistent_peers` değerini iki ayrı yerde yazıyor ve ikincisi birincisini eziyor. Yukarıdaki satır iki peer'ı birleştiriyor.

Sonucu doğrula:

```bash
grep -nE '^(proxy_app|pprof_laddr|prometheus_listen_addr|external_address|persistent_peers)|^laddr' $CFG/config.toml
grep -nE '^node|^keyring-backend|^chain-id' $CFG/client.toml
grep -nE '^(minimum-gas-prices|pruning|pruning-keep-recent|pruning-interval) ' $CFG/app.toml
grep -nE '^address' $CFG/app.toml
```

`persistent_peers` içindeki portlar uzaktaki peer'ların kendi portlarıdır ve olduğu gibi kalır. Güvenlik duvarı kullanıyorsan `${P}656/tcp` portunu aç.

## 9. State sync

Upstream rehberdeki eski snapshot linki bayat. Onun yerine state sync kullan.

```bash
SNAP_RPC=https://t-latanda.rpc.utsa.tech:443
LATEST=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
TRUST_H=$((LATEST - 2000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$TRUST_H" | jq -r .result.block_id.hash)
echo $LATEST $TRUST_H $TRUST_HASH     # üç değer de dolu olmalı, "null" olmamalı

# senkron sırasında sadece state sync peer'ı
sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"d6217d85d3747ebbfbf75898bd4407da567f5291@65.108.206.118:60556\"|" $CFG/config.toml

sed -i -E -e "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true|" \
          -e "s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"|" \
          -e "s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$TRUST_H|" \
          -e "s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" \
          -e "s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $CFG/config.toml

grep -nE '^(enable|rpc_servers|trust_height|trust_hash|trust_period|persistent_peers) ' $CFG/config.toml
```

`enable = true` sadece bir kez, `[statesync]` altında görünmeli. Servisi başlat (adım 10) ve logları izle. Keşfedilen bir snapshot, `Applied snapshot chunk` satırları ve en sonda `Snapshot restored` görmelisin.

**İlk denememiz başarısız oldu.** Trust yüksekliği tepe bloktan sadece 100 blok aşağıdayken ve tam peer listesiyle state sync eski snapshot'ları buldu, light client de `context deadline exceeded` ile zaman aşımına uğradı. Tepe bloktan yaklaşık 2000 blok aşağıda bir trust yüksekliği ve tek bir state sync peer'ı ile çalıştı. Yine zaman aşımı görürsen bloğu tekrar çalıştır, herkese açık RPC bir süre yavaş olabilir.

Node senkron olunca (`catching_up: false`):

```bash
systemctl stop latandad
sed -i -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1false|" $CFG/config.toml
sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"483a8110c3cd93c8dd3801d935151e98656f5b67@168.231.67.201:26656,d6217d85d3747ebbfbf75898bd4407da567f5291@65.108.206.118:60556\"|" $CFG/config.toml
systemctl start latandad
```

## 10. systemd servisi

```bash
tee /etc/systemd/system/latandad.service > /dev/null <<EOF
[Unit]
Description=La Tanda node (cosmovisor)
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --home $HOME/.latanda
Restart=on-failure
RestartSec=3
LimitNOFILE=65535
Environment="DAEMON_NAME=latandad"
Environment="DAEMON_HOME=$HOME/.latanda"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF

grep -E 'ExecStart|User' /etc/systemd/system/latandad.service   # ExecStart yolu boş olmamalı
systemctl daemon-reload && systemctl enable latandad && systemctl start latandad
journalctl -u latandad -f -o cat
```

`UNSAFE_SKIP_BACKUP=true`, upgrade sırasında data klasörünün kopyalanmasını atlar. Ekstra güvenlik istiyorsan ve disk alanın varsa kaldır. Logları yapıştırırken `journalctl ... --no-pager` kullan, böylece sayfalayıcı bir sonraki komutunu yutmaz.

Senkron durumunu kontrol et:

```bash
curl -s localhost:${P}657/status | jq '.result.sync_info | {latest_block_height, catching_up}'
```

Sadece `catching_up` değeri `false` olunca devam et.

## 11. Cüzdan ve faucet

Validatör için **yeni** bir cüzdan oluştur:

```bash
latandad keys add $WALLET
latandad keys show $WALLET -a      # ltd1... adresin
```

`file` keyring backend bir parola ister, onu sakla. Mnemonic'i çevrimdışı bir yere yaz.

Mevcut bir cüzdanı `latandad keys add $WALLET --recover` ile içeri aktarmak mümkün, ama gerçek bakiyesi olan bir cüzdanı aktarmamanı öneririz. Binary, kimsenin denetlemediği bir kaynak tarball'ından derleniyor ve testnet için bir test cüzdanı yeterli. Çıkan adres diğer Cosmos cüzdanındakinden farklıysa zincir farklı bir coin type kullanıyordur, sorun değil.

**Faucet.** Projeye göre faucet, captcha'yı geçtikten sonra Discord'daki `#node-onboarding` kanalından kişi başı bir kez 10 LTD veriyor. O kanalda sabitlenmiş istek formatını aynen kullan. Göndermeden önce adresi bir kez daha kontrol et, tek hakkın var.

```bash
latandad q bank balances $(latandad keys show $WALLET -a)
```

Faucet'ten sonra beklenen bakiye: `10000000ultd`.

## 12. Validatörü oluştur

Node senkron olana ve bakiye gelene kadar bekle.

```bash
cat > $HOME/.latanda/validator.json <<EOF
{
  "pubkey": $(latandad comet show-validator),
  "amount": "1000000ultd",
  "moniker": "$MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "<kisa aciklama>",
  "commission-rate": "0.1",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF

grep '"amount"' $HOME/.latanda/validator.json

latandad tx staking create-validator $HOME/.latanda/validator.json \
  --from $WALLET --chain-id $CHAIN_ID --fees 5000ultd -y
```

Tutar `ultd` cinsinden. `1000000ultd` 1 LTD eder. Bakiye tutar artı ücreti karşılamalı.

### Yerel çıktıya güvenme

Komut sonucu hemen yazdırır. `code: 0` ile birlikte `height: 0` ve `gas_used: 0` görmen sadece yerel yayın yanıtıdır, zincirin onayı değildir. Her zaman hash ile kontrol et:

```bash
latandad query tx --type=hash <TX_HASH> --output json | jq '{code, height, raw_log}'
```

İşlem sadece `code` değeri `0` ve `height` değeri 0'dan büyükse başarılıdır. Bizim durumumuzda ilk deneme yerelde `code: 0` yazdırdı, ama zincir sonucu `code: 5` ve `insufficient funds` idi, çünkü tutar 18 ondalıkla yazılmıştı. Başarısız işlem yine de 5000ultd ücreti yaktı. Böyle olursa tutarı düzeltip yeniden gönder:

```bash
sed -i 's/"amount": "[^"]*"/"amount": "1000000ultd"/' $HOME/.latanda/validator.json
grep '"amount"' $HOME/.latanda/validator.json
```

Validatörün oluştuğunu ve bonded olduğunu doğrula:

```bash
latandad q staking validator $(latandad keys show $WALLET --bech val -a)
latandad q slashing signing-info $(latandad comet show-validator)
```

`status: BOND_STATUS_BONDED` ve `jailed: false` görmelisin.

## 13. Discord doğrulaması

Projenin Discord botu iki şeyi kontrol ediyor: anahtar kontrolü kanıtı (`!verify`) ve altyapı kanıtı (HTTP-nonce ya da reverse DNS). Tam adımları, memo ve nonce değerlerini bot sana verir, aşağıdaki değerler yer tutucudur.

### 13.1 Anahtar kontrolü (`!verify`)

Bot sana bir memo verir. Validatör cüzdanından kendi adresine, o memo ile tam olarak 1 `ultd` gönder (self-send):

```bash
ADDR=$(latandad keys show $WALLET -a)
latandad tx bank send $WALLET $ADDR 1ultd \
  --note "<BOTUN_VERDIGI_MEMO>" \
  --chain-id $CHAIN_ID --fees 5000ultd -y
```

Gördüğümüz bot talimatlarında tam CLI komutu yoktu, bu standart Cosmos SDK biçimidir. Bot farklı söylüyorsa ona göre uyarla. İşlemi adım 12'deki gibi hash ile kontrol et, sonra doğrulama kanalına yaz:

```
!verify <TX_HASH>
```

Memo harfi harfine aynı olmalı.

### 13.2 Altyapı kanıtı (HTTP-nonce)

Bot ya reverse DNS kaydı ya da HTTP nonce ister. HTTP yolu için nonce'un sunucunun genel IP'sinin 80. portundan erişilebilir olması gerekir.

Önce 80. portu neyin kullandığına bak:

```bash
curl -4 -s https://api.ipify.org; echo
ss -ltnp | grep ':80 '
```

80. portu Nginx gibi bir web sunucusu zaten dinliyorsa **onu durdurma**. Token'ı onun web kök dizinine koy:

```bash
mkdir -p /var/www/html/.well-known
printf '%s' '<BOTUN_VERDIGI_NONCE>' > /var/www/html/.well-known/latanda-verify.txt

curl -i http://127.0.0.1/.well-known/latanda-verify.txt
curl -i http://<SUNUCU_IP>/.well-known/latanda-verify.txt
```

İki istek de `HTTP/1.1 200 OK` ve birebir token döndürmeli, fazladan satır sonu olmamalı (`printf '%s'` bunu halleder). Sonra Discord'da kontrolü çalıştır:

```
!verify-http-check
```

Botun verdiği yol ve dosya adını kullan, yukarıdakiler bizden istenenlerdi. Bot kaldırmanı söylemedikçe dosyayı yerinde bırak. 80. portta hiçbir şey çalışmıyorsa ve web sunucun yoksa dosyayı geçici ve minimal bir HTTP sunucusuyla sunmak mümkün olabilir, bunu denemedik. Reverse DNS alternatifi hosting sağlayıcından bir PTR kaydı gerektirir, bu da burada denenmedi.

### 13.3 Tier kontrol listesi

Bot şuna benzer bir kontrol listesi bildiriyor. Bu, Ekim 2026'da gördüğümüz hali ve program değişebilir:

| Koşul | Anlamı |
| --- | --- |
| Validatör aktif, bonded, jail dışı | `create-validator` başarılı oldu ve node imza atıyor |
| `!verify` (anahtar kontrolü) | botun memo'suyla self-send |
| Altyapı kanıtı | HTTP-nonce ya da rDNS |
| Jail olmadan 7 gün imzalama | kurulumdan hemen sonra `0/7` görünüyordu, beklemen gerekir |
| Tier delegasyonu | ilk dört koşuldan sonra değerlendirilir |

Ekibin forum yazısına göre destek delegasyonları ekibin imzaladığı gruplar halinde gidiyor, hiçbir şey otomatik değil ve testnet LTD'nin parasal değeri yok.

## 14. İzleme

Sunucunun açık olması yetmez. Validatörün gerçekten blok imzalaması gerekir.

```bash
# node sağlığı
curl -s localhost:${P}657/status | jq '.result.sync_info | {latest_block_height, catching_up}'

# validatör durumu
latandad q staking validator $(latandad keys show $WALLET --bech val -a) | grep -E 'status|jailed|tokens'

# imza ve jail bilgisi
latandad q slashing signing-info $(latandad comet show-validator)

# loglar
journalctl -u latandad -n 50 --no-pager -o cat
```

`missed_blocks_counter` ve `jailed_until` değerlerini izle. Bir yönetişim önerisi bootstrap döneminde `signed_blocks_window` değerini 100'den 10.000'e çıkardı, bu yüzden kısa kesintiler seni jail'e düşürme ihtimali daha düşük, ama uzun bir kesinti yine düşürebilir. Jail'e düşersen önce node'u düzelt, sonra:

```bash
latandad tx slashing unjail --from $WALLET --chain-id $CHAIN_ID --fees 5000ultd -y
```

Node'u uzun aralıklar olmadan çalışır tut. Tier koşulu jail olmadan 7 gün, yani bir jail en azından süreyi geciktirir.

## 15. Cosmovisor ile güncelleme

Yeni binary'yi derle ve bir upgrade klasörüne koy:

```bash
cd $HOME/latanda-build
wget -q https://latanda.online/chain/latanda-chain-source.tar.gz && tar -xzf latanda-chain-source.tar.gz
go build -o latandad ./cmd/latandad
mkdir -p $HOME/.latanda/cosmovisor/upgrades/<UPGRADE_ADI>/bin
cp latandad $HOME/.latanda/cosmovisor/upgrades/<UPGRADE_ADI>/bin/
```

Sonra iki durumdan biri:

- **Upgrade zincir üstü bir upgrade planıysa.** `<UPGRADE_ADI>` plan adıyla birebir aynı olmalı. Cosmovisor upgrade yüksekliğinde kendisi geçer.
- **Ağ bir blokta duruyor ve operatörler binary'yi elle değiştiriyorsa.** Elle geç:

```bash
systemctl stop latandad
ln -sfn $HOME/.latanda/cosmovisor/upgrades/<UPGRADE_ADI> $HOME/.latanda/cosmovisor/current
systemctl start latandad
```

Ekibin hangisini kullandığını bilmiyoruz. `latandad q upgrade plan` komutu upgrade modülünün aktif olup olmadığını gösterir, upgrade'lerin nasıl koordine edildiğini ise ekibin teknik kanalı doğrulayabilir.

## 16. Yedekleme ve güvenlik

```bash
mkdir -p $HOME/latanda-backup
cp $CFG/priv_validator_key.json $CFG/node_key.json $HOME/latanda-backup/
```

Bu dosyaları ve cüzdan mnemonic'ini şifreli olarak **sunucunun dışına** taşı. Sunucu diski bozulursa ve kopyan yoksa validatör kimliği kaybolur.

- Aynı `priv_validator_key.json` dosyasını aynı anda iki makinede çalıştırma, bu double sign demektir.
- Mevcut bir validatörde snapshot'tan geri yüklüyorsan kendi `priv_validator_state.json` dosyanı koru.
- Seed phrase'i ya da özel anahtarı Discord'a, herhangi bir asistana ya da bota asla gönderme.
- Testnet için ayrı bir cüzdan kullan.
- HTTP kontrolünü geçmek için mevcut web servisini durdurma.

## 17. Sorun giderme

| Belirti | Sebep ve çözüm |
| --- | --- |
| `create-validator` `code: 0` yazdırıyor ama validatör yok | Çıktı sadece yereldi. İşlemi hash ile sorgula. `code` ve `height` değerlerine bak. |
| Zincirde `insufficient funds` | Tutar 18 ondalıkla yazılmış. `1000000ultd` (1 LTD) kullan. |
| CLI sorguları başka bir zincirin verisini döndürüyor | `client.toml` içindeki `node` hâlâ 26657'yi gösteriyor. `tcp://localhost:${P}657` yap. |
| State sync: `context deadline exceeded`, eski snapshot'lar | Tepe bloktan yaklaşık 2000 blok aşağıda bir trust yüksekliği, tek bir state sync peer'ı kullan ve tekrar dene. |
| Genesis hash uyuşmuyor | Devam etme. Discord'dan güncel hash'i sor. |
| `ss` portlarını kullanımda gösteriyor | Başka bir `P` öneki seç ve adım 8'i tekrarla. |
| Servis dosyasında `ExecStart` boş | `cosmovisor` PATH'te değil. Dosyayı sil, `which cosmovisor` bir yol verdikten sonra yeniden oluştur. |
| Discord daveti çalışmıyor ya da hesap atılıyor | Sunucunun bot filtresi captcha'yı geçemeyen hesapları kaldırıyor, ekip de elle whitelist yapmadığını söylemiş. |
| `!verify` başarısız | Memo birebir değil, yanlış tx hash'i ya da işlem zincire girmemiş. Hash ile kontrol et. |
| `!verify-http-check` başarısız | 80. porta internetten erişilemiyor, yanlış dosya yolu, token'da fazladan satır sonu ya da güvenlik duvarı. Genel IP'ye `curl` ile test et. |
| Tier kurulumdan hemen sonra `0/7` gösteriyor | Beklenen durum. Validatörü bonded, imzalayan ve jail dışı tut, sonra tekrar bak. |

## 18. Kaynaklar

- La Tanda için UTSA topluluk rehberi (temel adımlar, addrbook, state sync endpoint'i)
- La Tanda ekibinin forum yazıları (zincir bilgileri, destek programı, tier kuralları)
- Yukarıdaki sorunları test edip bildiren herkes

Pull request ve düzeltmeler memnuniyetle karşılanır.
