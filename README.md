# base221tpimport time
from collections import defaultdict
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 20
REPEAT_THRESHOLD = 5


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect to Base RPC")

    print("Connected to Base")
    print("Detecting possible wash trading...\n")

    last_block = w3.eth.block_number

    while True:
        try:
            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                from_block = current_block - WINDOW_BLOCKS
                to_block = current_block

                logs = w3.eth.get_logs({
                    "fromBlock": from_block,
                    "toBlock": to_block,
                    "topics": [TRANSFER_TOPIC]
                })

                interaction_counter = defaultdict(int)

                for log in logs:

                    token = log["address"]

                    from_addr = decode_address(log["topics"][1])
                    to_addr = decode_address(log["topics"][2])

                    if from_addr == to_addr:
                        continue

                    pair_key = (
                        token,
                        tuple(sorted([from_addr, to_addr]))
                    )

                    interaction_counter[pair_key] += 1

                print(f"\nBlocks {from_block} → {to_block}")

                for pair_key, count in interaction_counter.items():

                    if count >= REPEAT_THRESHOLD:

                        token, wallets = pair_key

                        print("🌀 Possible Wash Trading")
                        print("Token:", token)
                        print("Wallet A:", wallets[0])
                        print("Wallet B:", wallets[1])
                        print("Interactions:", count)
                        print()

                last_block = current_block

            time.sleep(3)

        except Exception as e:
            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
