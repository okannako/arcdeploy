<p align="center">
  <a href="">
    <img alt="Hero" src= "https://github.com/user-attachments/assets/e8db31b0-2ddc-4254-8a1c-416eda010214" />
  </a>
</p>

## Arc Testnet’e İlk Sözleşme Deploy Rehberi (Foundry + Ubuntu 22.04)
- Bu rehber, Arc resmi “Deploy on Arc” dokümanı başta olmak üzere Arc Docs’tan derlenmiştir. Amacımız sıfırdan Ubuntu 22.04 üzerinde Foundry ile Arc Testnet’e basit bir sözleşme deploy edip etkileşim kurmaktır.

### İçindekiler

- Arc Testnet Bağlantı Bilgileri
- Ön Koşullar
- Sistem Bağımlılıkları ve Foundry Kurulumu (Ubuntu 22.04)
- Foundry Projesi Oluştur ve Arc RPC’yi Yapılandır
- HelloArchitect Sözleşmesini Yaz
- Test ve Derleme
- Cüzdan Oluştur, Testnet USDC Al ve Deploy Et
- Etkileşim: Explorer’dan Kontrol ve Cast ile Okuma

1️⃣ Arc Testnet Bağlantı Bilgileri
- Bu değerler kurulum boyunca kullanılacak (resmi Connect to Arc sayfasından).
- Ağ adı: Arc Testnet
- RPC endpoint: https://rpc.testnet.arc.network
- Chain ID: 5042002
- Para birimi (gas): USDC
- Block explorer: https://testnet.arcscan.app
- Faucet: https://faucet.circle.com (USDC almak için, ağ seçimi: Arc Testnet)

2️⃣ Ön Koşullar
- Makine: Ubuntu 22.04 (sunucu/masaüstü farketmez)
- Kullanıcı: sudo yetkili bir kullanıcı ya da root
- Ağ: HTTPS ile dışarıya çıkabilen bir bağlantı (foundryup ve RPC için)
- Cüzdan: İstersen tarayıcıda MetaMask kur; ancak bu rehber Foundry’nin cast wallet new ile CLI cüzdan kullanır. Metamask gerekli değil ama cüzdanı görmek isterseniz import edebilirsiniz.
- Hesap: Circle Faucet’ten testnet USDC alabilmek için bir cüzdan adresi (CLI ile oluşturacağız)

3️⃣ Sistem Bağımlılıkları ve Foundry Kurulumu (Ubuntu 22.04)
- Foundry’nin kendisi kurulumu şu iki komuttan ibaret olsa da, Ubuntu’da önce temel build araçlarını kurmak daha sağlıklı. 

3.1) Sistem paketlerini güncelle ve temel bağımlılıkları kur.
```
sudo apt update
sudo apt install -y build-essential curl git ca-certificates
```

3.2) foundryup betiğini indir ve yükle.
- Resmi kurulum (Foundry README’si ve Arc dokümanıyla aynı akış): 
````
curl -L https://foundry.paradigm.xyz | bash
````
- Çıktıda, foundryup’un PATH’e eklendiği söylenecektir (genellikle ~/.bashrc veya benzeri). Etkinleştirmek için:
````
source ~/.bashrc   # veya kullandığınız shell’e göre: source ~/.zshrc
````

3.3) Foundry araçlarını (forge, cast, anvil, chisel) kur.
- Alttaki kodla kurulum yapıyoruz.
````
foundryup
````

3.4) Kurulumu doğrula.
````
forge --version
cast --version
anvil --version
chisel --version
````
- En azından forge ve cast versiyonları görünmelidir.

4️⃣ Foundry Projesi Oluştur ve Arc RPC’yi Yapılandır
- Bu bölüm yine birebir Arc dokümanından alınmıştır.

4.1) Yeni bir Foundry projesi başlat.
````
forge init hello-arc && cd hello-arc
````
- Bu, klasik Counter.sol şablonu ile birlikte src/, script/, test/ vb. klasörleri oluşturur.

4.2) Arc RPC için .env dosyası oluştur.
- Arc, .env dosyasında RPC URL’sini saklamayı öneriyor.

- Proje kökünde (hello-arc/):
````
nano .env
````
- .env içine şunu ekle ( RPC, Arc Connect sayfasından):
````
# Arc Testnet RPC
ARC_TESTNET_RPC_URL="https://rpc.testnet.arc.network"

# Birazdan deploy için kullanacağız:
# PRIVATE_KEY="0x..."           # ASLA gerçek projede plain text .env'e yazma; bu rehber sadece test için
# HELLOARCHITECT_ADDRESS="0x..." # Deploy sonrası doldurulacak
````

5️⃣ HelloArchitect Sözleşmesini Yaz
- Aşağıdaki adımlar birebir Arc dokümanındandır; sadece dosya yolları Ubuntu’ya uygun gösterilmiştir.

5.1) Varsayılan Counter.sol’u sil.
````
ls -R test/ | grep -i counter
rm src/Counter.sol
forge clean
````
5.2) src/HelloArchitect.sol oluştur ve aşağıdaki kodu yapıştır.
````
nano src/HelloArchitect.sol
````
- İçerik (dokümandaki haliyle):
````
solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

contract HelloArchitect {
    string private greeting;

    // Event emitted when the greeting is changed
    event GreetingChanged(string newGreeting);

    // Constructor that sets the initial greeting to "Hello Architect!"
    constructor() {
        greeting = "Hello Architect!";
    }

    // Setter function to update the greeting
    function setGreeting(string memory newGreeting) public {
        greeting = newGreeting;
        emit GreetingChanged(newGreeting);
    }

    // Getter function to return the current greeting
    function getGreeting() public view returns (string memory) {
        return greeting;
    }
}
````

6️⃣ Test ve Derleme
6.1) Eski script ve test dosyalarını temizle.
- Arc dokümanı, Counter.sol referanslı script/test dosyalarını kaldırmamızı öneriyor.
```
rm -rf script
````
6.2) Yeni test dosyasını oluştur.
````
nano test/HelloArchitect.t.sol
````
- Aşağıdaki kodu test/HelloArchitect.t.sol dosyasına yapıştır (dokümandaki haliyle):
````
solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import "forge-std/Test.sol";
import "../src/HelloArchitect.sol";

contract HelloArchitectTest is Test {
    HelloArchitect helloArchitect;

    function setUp() public {
        helloArchitect = new HelloArchitect();
    }

    function testInitialGreeting() public view {
        string memory expected = "Hello Architect!";
        string memory actual = helloArchitect.getGreeting();
        assertEq(actual, expected);
    }

    function testSetGreeting() public {
        string memory newGreeting = "Welcome to Arc Chain!";
        helloArchitect.setGreeting(newGreeting);
        string memory actual = helloArchitect.getGreeting();
        assertEq(actual, newGreeting);
    }

    function testGreetingChangedEvent() public {
        string memory newGreeting = "Building on Arc!";
        // Expect the GreetingChanged event to be emitted
        vm.expectEmit(true, true, true, true);
        emit HelloArchitect.GreetingChanged(newGreeting);
        helloArchitect.setGreeting(newGreeting);
    }
}
````

6.3) Testleri çalıştır.
````
forge test
````
- Bütün testler geçmelidir.

6.4) Sözleşmeyi derle.
````
forge build
````
- Bu, out/ klasöründe bytecode ve ABI üretir; deploy sırasında Foundry bunları kullanır.

7️⃣ Cüzdan oluştur, testnet USDC al ve deploy et
- Bu bölüm yine Arc dokümanının “Deploy your contract to Arc testnet” kısmına birebir uygundur.

7.1) Yeni bir cüzdan oluştur.
````
cast wallet new
````
- Çıktı şuna benzer (doküman örnek çıktısı):
````
Successfully created new keypair.
Address:     0xB815A0c4bC23930119324d4359dB65e27A846A2d
Private key: 0xcc1b30a6af68ea9a9917f1dd.......................................97c5
````
- Address → bu adrese testnet USDC göndereceğiz
- Private key → deploy için kullanacağız (testnet olduğu için bu rehberde .env’e alacağız; üretimde asla böyle yapma)

7.2) .env dosyasını güncelle.
- .env dosyasına az önceki PRIVATE_KEY değerini ekle:
```
# Arc Testnet RPC
ARC_TESTNET_RPC_URL="https://rpc.testnet.arc.network"

# Deploy için (SADECE TESTNET)
PRIVATE_KEY="0xcc1b30a6af68ea9a9917f1dd....97c5"  # burayı kendi private keyinle değiştir

# Deploy sonrası doldurulacak
# HELLOARCHITECT_ADDRESS="0x...."
````
- Kaydet:
````
source .env
````

7.3) Cüzdana testnet USDC gönder.
- cast wallet new çıktısındaki Address: değerini kopyala.
- Tarayıcıdan Faucet’e git: https://faucet.circle.com
- Ağ olarak “Arc Testnet” seç (veya adresi girdiğinde otomatik tanır).
- Adresi yapıştır ve testnet USDC talep et.
- USDC, Arc’te native gas tokenı olduğu için, bu bakiye deploy sırasında gas ödemek için yeterlidir.

7.4) Sözleşmeyi Arc Testnet’e deploy et.
- Arc dokümanındaki deploy komutu (çok küçük bir formatting uyarlamasıyla):
````
forge create src/HelloArchitect.sol:HelloArchitect \
  --rpc-url $ARC_TESTNET_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast
````
- Başarılı olursa çıktı şuna benzer:
````
Compiler run successful!
Deployer:      0xB815A0c4bC23930119324d4359dB65e27A846A2d
Deployed to:   0x32368037b14819C9e5Dbe96b3d67C59b8c65c4BF
Transaction hash: 0xeba0fcb5e528d586db0aeb2465a8fad0299330a9773ca62818a1827560a67346
````
- Deployed to: satırındaki adresi kaydet.

7.5) Sözleşme adresini .env’e kaydet ve yeniden yükle.
````
# ...
HELLOARCHITECT_ADDRESS="0x32368037b14819C9e5Dbe96b3d67C59b8c65c4BF"  # kendi adresinle değiştir
````
- Sonra tekrar:
````
source .env
````

8️⃣ Etkileşim: Explorer’dan Kontrol ve Cast ile Okuma
8.1) Explorer’da kontrol.
- Arc Testnet Explorer’a git: https://testnet.arcscan.app
- Arama kutusuna deploy çıktısındaki Transaction hash değerini yapıştır.
- Detayları görüntüle: durum (status), from/to (contract creation), vs.

8.2) cast ile getGreeting() çağrısı.
- Arc dokümanındaki örneği aynen uygulayabiliriz:
```
cast call $HELLOARCHITECT_ADDRESS "getGreeting()(string)" \
  --rpc-url $ARC_TESTNET_RPC_URL
````
- Dönen değer (decode edilmiş string):
```
Hello Architect!
````
- Bu, sözleşmenin canlıda çalıştığını gösterir.

8.3) (İsteğe bağlı) cast send ile setGreeting() çağrısı.
- Örnek (USDC bakiyeniz olduğunda gas ödenecektir):
````
cast send $HELLOARCHITECT_ADDRESS \
  "setGreeting(string)" \
  "Selam Arc!" \
  --rpc-url $ARC_TESTNET_RPC_URL \
  --private-key $PRIVATE_KEY
````
- Sonra tekrar:
````
cast call $HELLOARCHITECT_ADDRESS "getGreeting()(string)" \
  --rpc-url $ARC_TESTNET_RPC_URL
````
- Artık ````Selam Arc!```` dönmesi gerekir.

Not: Kullandığım resmi kaynaklar aşağıdadır. Daha ayrıntılı adımları linklerde bulabilirsiniz.

- Deploy on Arc (Foundry ile): https://docs.arc.network/arc/tutorials/deploy-on-arc
- Connect to Arc (RPC, Chain ID, Explorer, Faucet): https://docs.arc.network/arc/references/connect-to-arc
- Foundry resmi kurulum: https://github.com/foundry-rs/foundry
