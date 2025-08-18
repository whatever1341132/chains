import json
import glob
from web3 import Web3
from mnemonic import Mnemonic
from bip44 import Wallet
from binance.client import Client
import cbpro

# ===============================
# 3️⃣ Config / Secrets
# ===============================
with open("config/secrets.json") as f:
    config = json.load(f)

INFURA_ID = config["INFURA_PROJECT_ID"]
SEED_PHRASE = config["SEED_PHRASE"]
BINANCE_API_KEY = config["BINANCE_API_KEY"]
BINANCE_API_SECRET = config["BINANCE_API_SECRET"]
COINBASE_API_KEY = config["COINBASE_API_KEY"]
COINBASE_API_SECRET = config["COINBASE_API_SECRET"]
COINBASE_PASSPHRASE = config["COINBASE_PASSPHRASE"]

# ===============================
# 4️⃣ Wallets: Seed phrase -> Addresses
# ===============================
def derive_addresses(seed_phrase, num_addresses=5):
    mnemo = Mnemonic("english")
    if not mnemo.check(seed_phrase):
        raise ValueError("Invalid seed phrase")
    wallet = Wallet(seed_phrase)
    return [wallet.get_address("eth", account=i) for i in range(num_addresses)]

addresses = derive_addresses(SEED_PHRASE)
print(f"Derived addresses: {addresses}\n")

# ===============================
# 5️⃣ Load EVM Chains
# ===============================
def load_chains(path="_data/chains/*.json"):
    chains = []
    for file in glob.glob(path):
        with open(file, "r") as f:
            data = json.load(f)
            required_keys = ["name", "chain", "rpc", "nativeCurrency", "chainId"]
            if not all(k in data for k in required_keys):
                continue
            chains.append(data)
    return chains

def get_balance(address, chain, infura_id=None):
    for rpc in chain["rpc"]:
        try:
            if "${INFURA_API_KEY}" in rpc and infura_id:
                rpc = rpc.replace("${INFURA_API_KEY}", infura_id)
            w3 = Web3(Web3.HTTPProvider(rpc))
            if w3.is_connected():
                balance = w3.eth.get_balance(address)
                return w3.from_wei(balance, "ether")
        except Exception:
            continue
    return None

chains = load_chains()
print(f"Loaded {len(chains)} EVM chains.\n")

# ===============================
# 6️⃣ Exchanges: Binance & Coinbase
# ===============================
# Binance
binance_client = Client(BINANCE_API_KEY, BINANCE_API_SECRET)
def get_binance_eth_price(client):
    return float(client.get_symbol_ticker(symbol="ETHUSDT")["price"])

# Coinbase
coinbase_client = cbpro.AuthenticatedClient(COINBASE_API_KEY, COINBASE_API_SECRET, COINBASE_PASSPHRASE)
def get_coinbase_eth_price(client):
    return float(client.get_product_ticker(product_id="ETH-USD")["price"])

# ===============================
# 7️⃣ Multi-chain Balances
# ===============================
print("=== MULTI-CHAIN BALANCES ===")
for chain in chains:
    for addr in addresses:
        balance = get_balance(addr, chain, INFURA_ID)
        symbol = chain["nativeCurrency"]["symbol"]
        print(f"[{chain['name']}] Address: {addr} | Balance: {balance} {symbol}")

# ===============================
# 8️⃣ Price & Profit Simulation
# ===============================
binance_eth_price = get_binance_eth_price(binance_client)
coinbase_eth_price = get_coinbase_eth_price(coinbase_client)
print(f"\nETH price on Binance: {binance_eth_price}")
print(f"ETH price on Coinbase: {coinbase_eth_price}")

print("\n=== PROFIT SIMULATION ===")
if binance_eth_price < coinbase_eth_price:
    profit_percent = (coinbase_eth_price - binance_eth_price) / binance_eth_price * 100
    print(f"Opportunity: Buy ETH on Binance, sell on Coinbase, profit ~{profit_percent:.2f}%")
else:
    print("No profitable arbitrage opportunity currently.")

print("\n[INFO] Aggregation complete.")