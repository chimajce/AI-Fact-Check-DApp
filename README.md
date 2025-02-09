from flask import Flask, request, jsonify
import requests
from autonomys_sdk import AutonomysStorage  # Hypothetical SDK import
from web3 import Web3
import openai

app = Flask(__name__)

# Connect to blockchain (Ethereum/Polygon)
w3 = Web3(Web3.HTTPProvider("https://polygon-rpc.com"))  # Replace with actual RPC
contract_address = "0xYourSmartContractAddress"
contract_abi = "[...]"  # Load ABI from compiled contract
contract = w3.eth.contract(address=contract_address, abi=contract_abi)

# OpenAI API key (Replace with your own key)
openai.api_key = "your-openai-key"

# Autonomys decentralized storage setup (hypothetical usage)
autonomys_storage = AutonomysStorage(api_key="your-autonomys-api-key")

@app.route("/fact-check", methods=["POST"])
def fact_check():
    data = request.json
    article_url = data.get("url")
    if not article_url:
        return jsonify({"error": "URL required"}), 400
    
    # Fetch article text (simplified, use proper scraping in production)
    response = requests.get(article_url)
    article_text = response.text[:1000]  # Limit for testing
    
    # AI Fact-Checking Analysis
    ai_response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": "Analyze credibility of news articles based on verifiable sources."},
            {"role": "user", "content": article_text}
        ]
    )
    credibility_score = ai_response["choices"][0]["message"]["content"]
    
    # Store results on Autonomys decentralized storage
    storage_data = {
        "url": article_url,
        "score": credibility_score,
        "ai_verdict": "Reliable" if "high" in credibility_score else "Unreliable"
    }
    tx_hash = autonomys_storage.store_data(storage_data)
    
    # Store reference on-chain (Ethereum/Polygon)
    tx = contract.functions.storeFactCheck(article_url, credibility_score).transact({"from": w3.eth.accounts[0]})
    w3.eth.wait_for_transaction_receipt(tx)
    
    return jsonify({"message": "Fact check completed!", "storage_tx": tx_hash, "on_chain_tx": tx.hex()})

if __name__ == "__main__":
    app.run(debug=True)
