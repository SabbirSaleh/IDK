pipeline {
    agent any

    environment {
    FABRIC_CFG_PATH = "${WORKSPACE}/config"  // Path to config directory
    CHAINCODE_NAME = "basic"                // Name of chaincode
    CHAINCODE_VERSION = "1.0"               // Chaincode version
    CHANNEL_NAME = "mychannel"              // Channel name
    PEER0_ORG1_TLSCERT = "${WORKSPACE}/test-network/organizations/peerOrganizations/org1.example.com/tlsca/tlsca.org1.example.com-cert.pem"
    ORDERER_TLSCERT = "${WORKSPACE}/test-network/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem"
    PATH = "${WORKSPACE}/bin:${PATH}"       // Add binaries from the repository to PATH
}


    stages {
        stage('Setup Environment') {
            steps {
                sh '''
                echo "Setting up environment..."
                docker --version
                peer version || echo "Peer binary not found"
                '''
            }
        }

        stage('Start Fabric Network') {
            steps {
                sh '''
                echo "Starting Fabric test network..."
                cd test-network
                ./network.sh down
                ./network.sh up createChannel -c ${CHANNEL_NAME}
                ./network.sh deployCC -ccn ${CHAINCODE_NAME} -ccp ../asset-transfer-basic/chaincode-go -ccl go
                '''
            }
        }

        stage('Install Chaincode') {
            steps {
                sh '''
                echo "Installing chaincode..."
                export CORE_PEER_LOCALMSPID="Org1MSP"
                export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG1_TLSCERT}
                export CORE_PEER_MSPCONFIGPATH=${WORKSPACE}/test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
                export CORE_PEER_ADDRESS=localhost:7051

                peer lifecycle chaincode install ${CHAINCODE_NAME}.tar.gz
                '''
            }
        }

        stage('Approve Chaincode for Org1') {
            steps {
                sh '''
                echo "Approving chaincode for Org1..."
                export CORE_PEER_LOCALMSPID="Org1MSP"
                export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG1_TLSCERT}
                export CORE_PEER_MSPCONFIGPATH=${WORKSPACE}/test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
                export CORE_PEER_ADDRESS=localhost:7051

                peer lifecycle chaincode approveformyorg -o localhost:7050 \
                    --ordererTLSHostnameOverride orderer.example.com \
                    --tls --cafile ${ORDERER_TLSCERT} \
                    --channelID ${CHANNEL_NAME} \
                    --name ${CHAINCODE_NAME} \
                    --version ${CHAINCODE_VERSION} \
                    --sequence 1
                '''
            }
        }

        stage('Commit Chaincode') {
            steps {
                sh '''
                echo "Committing chaincode..."
                peer lifecycle chaincode commit -o localhost:7050 \
                    --ordererTLSHostnameOverride orderer.example.com \
                    --tls --cafile ${ORDERER_TLSCERT} \
                    --channelID ${CHANNEL_NAME} \
                    --name ${CHAINCODE_NAME} \
                    --peerAddresses localhost:7051 --tlsRootCertFiles ${PEER0_ORG1_TLSCERT} \
                    --version ${CHAINCODE_VERSION} --sequence 1
                '''
            }
        }

        stage('Invoke Chaincode') {
            steps {
                sh '''
                echo "Invoking chaincode..."
                peer chaincode invoke -o localhost:7050 \
                    --ordererTLSHostnameOverride orderer.example.com \
                    --tls --cafile ${ORDERER_TLSCERT} \
                    -C ${CHANNEL_NAME} -n ${CHAINCODE_NAME} \
                    --peerAddresses localhost:7051 --tlsRootCertFiles ${PEER0_ORG1_TLSCERT} \
                    -c '{"Args":["InitLedger"]}'
                '''
            }
        }

        stage('Query Chaincode') {
            steps {
                sh '''
                echo "Querying chaincode..."
                peer chaincode query -C ${CHANNEL_NAME} -n ${CHAINCODE_NAME} -c '{"Args":["QueryAllAssets"]}'
                '''
            }
        }
        stage('Download Binaries') {
            steps {
                sh '''
                curl -sSL https://bit.ly/2ysbOFE | bash -s -- -d
                mv bin/* ${WORKSPACE}/bin/
                export PATH=${WORKSPACE}/bin:$PATH
                '''
            }    
        }

    }

    post {
        success {
            echo 'Pipeline execution succeeded!'
        }
        failure {
            echo 'Pipeline execution failed.'
        }
    }
}

