h1 auto_gravity_cron

#!/bin/bash

# //////////// EDITABLE VARIABLES ////////////

DELEGATOR_ADDRESS='gravity1vgt7x7pln8f2uj8qpefeegxrfw90rxh3fc8whk'         # delegator address, without 'valoper'
VALIDATOR_ADDRESS='gravityvaloper1vgt7x7pln8f2uj8qpefeegxrfw90rxh3cn7saz'  # valoper address, with 'valoper'
WALLET_NAME='cyberomanov'                                                  # your keys name
PASSWORD='21092000'                                                        # your password from keys
COSMOS='/usr/bin/gravity'                                                  # full path to executable bin
CONFIG='/root/.gravity/config/'                                            # full path to config directory
KEYRING='os'                                                               # keyring keys type (os|file|kwallet|pass|test|memory)
FEE_AMOUNT=5000                                                            # fees for each transaction
TOKEN=ugraviton                                                            # native token to pay fees
DENOM=1000000                                                              # denomination
DELAY=10                                                                   # delay between cycles

# //////////// NOT EDITABLE VARIABLES ////////////

LRED='\033[0;31m'; LWHITE='\033[0m'; LCYAN='\033[1;36m'; LBLUE='\033[1;34m'; LMAGENTA='\033[1;35m'
NODE=`cat ${CONFIG}/config.toml | grep -oPm1 "(?<=^laddr = \")([^%]+)(?=\")"`
CHAIN=`cat ${CONFIG}/genesis.json | jq .chain_id | sed -E 's/.*"([^"]+)".*/\1/'`
SAFE_BALANCE=$((${FEE_AMOUNT} * 5))

# //////////// FUNCTIONS ////////////

# getting sequence to prevent sequence mismatch
function __getSequence() {
    expr `${COSMOS} query account ${DELEGATOR_ADDRESS} --chain-id ${CHAIN} --node ${NODE} -o json | jq ".base_account.sequence" | bc`
}

# getting current rewards
function __getRewards() {
    FLOAT_REWARDS=$(${COSMOS} query distribution validator-outstanding-rewards ${VALIDATOR_ADDRESS} --chain-id ${CHAIN} --node ${NODE} --output json | jq '.rewards[] .amount' | bc)
    REWARDS=$(echo "scale=5;${FLOAT_REWARDS%.*} / ${DENOM}" | bc )
    if (( ${#REWARDS} < 7 )); then REWARDS="0${REWARDS}"; fi
    echo ${REWARDS}
}

# withdrawing all we can
function withdrawAllRewardsFunc() {
    echo -e "total reward: `__getRewards` ${TOKEN}"
    TX=$(echo -e "${PASSWORD}\ny\n" | ${COSMOS} tx distribution withdraw-rewards ${VALIDATOR_ADDRESS} --commission --from ${WALLET_NAME} --chain-id ${CHAIN} --keyring-backend ${KEYRING} --fees 0${TOKEN} --node ${NODE} --sequence `__getSequence` -o json --yes | jq '.txhash' | tr -d '"')
    echo -e " "
    echo -e "withdrawing... tx: ${TX}"
	echo -e " "
    sleep 10
}

# delegating safe amount of tokens
function delegationFunc() {
    BALANCE=$(${COSMOS} query bank balances ${DELEGATOR_ADDRESS} --chain-id ${CHAIN} --node ${NODE} --output json | jq '.balances[] | select(.denom == "'"$TOKEN"'") .amount' | bc)
    if (( ${BALANCE} > ${SAFE_BALANCE} )); then
        BALANCE=`expr $BALANCE - ${SAFE_BALANCE}`
        BALANCE_TEXT=$(echo "scale=5;${BALANCE%.*} / ${DENOM}" | bc)
        if (( ${#BALANCE_TEXT} < 7 )); then BALANCE_TEXT="0${BALANCE_TEXT}"; fi
	    echo -e "delegation amount: ${BALANCE_TEXT} ${TOKEN}"
	    echo -e " "
        TX=$(echo -e "${PASSWORD}\n${PASSWORD}\n" | ${COSMOS} tx staking delegate ${VALIDATOR_ADDRESS} ${BALANCE}${TOKEN} --from=${WALLET_NAME} --fees ${FEE_AMOUNT}${TOKEN} --chain-id ${CHAIN} --keyring-backend ${KEYRING} --node ${NODE} --sequence `__getSequence` -o json --yes  | jq '.txhash' | tr -d '"')
        echo -e "delegating... tx: ${TX}"
	    echo -e " "
        sleep 10
    else
        echo -e "balance: ${BALANCE} ${TOKEN}, which is smaller than ${SAFE_BALANCE} ${TOKEN}"
	    echo -e " "
    fi
    BALANCE=$(${COSMOS} query bank balances ${DELEGATOR_ADDRESS} --chain-id ${CHAIN} --node ${NODE} --output json | jq '.balances[] | select(.denom == "'"$TOKEN"'") .amount' | bc)
    BALANCE_TEXT=$(echo "scale=5;${BALANCE%.*} / ${DENOM}" | bc)
    if (( ${#BALANCE_TEXT} < 7 )); then BALANCE_TEXT="0${BALANCE_TEXT}"; fi
    echo -e "balance: ${BALANCE_TEXT} ${TOKEN}"
}

# starting endless loop


echo -e "/// $(date) ///"
echo -e " "

withdrawAllRewardsFunc
delegationFunc

echo -e " "
