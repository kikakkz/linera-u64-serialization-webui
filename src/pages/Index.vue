<template>
  <div class='row'>
    <q-space />
    <div :style='{width: "600px", marginTop: "80px"}'>
      <q-input v-model='chainId' label='Input microchain ID' />
      <q-input v-model='nodeServiceUrl' label='Node service url' />
      <q-input v-model='faucetUrl' label='Faucet url' />
      <q-input v-model='publicKey' label='Public key' />
      <q-input v-model='privateKey' label='Private key' />
      <div :style='{height: "16px"}' />
      <q-btn label='OPEN CHAIN' @click='onOpenChainClick' :disable='!publicKey.length' />
      <div :style='{height: "16px"}' />
      <q-btn label='INITIALIZE WALLET WITH PUBLIC KEY' @click='onWalletInitWithPublicKeyClick' :disable='!publicKey.length' />
      <div :style='{height: "16px"}' />
      <q-btn label='GET BLOCK' @click='onGetBlockClick' :disable='!chainId.length' />
      <div :style='{height: "16px"}' />
      <q-btn label='GET BLOCK MATERIAL' @click='onGetBlockMaterialClick' :disable='!chainId.length' />
      <div :style='{height: "16px"}' />
      <q-btn label='EXECUTE BLOCK' @click='onExecuteBlockClick' />
    </div>
    <q-space />
  </div>
</template>

<script setup lang='ts'>
import axios from 'axios'
import { onMounted, ref } from 'vue'
import { gql } from '@apollo/client/core'
import { Ed25519SigningKey, Memory, Berith } from '@hazae41/berith'

const BLOCK = gql`
  query block($chainId: ChainId!, $hash: CryptoHash) {
    block(chainId: $chainId, hash: $hash) {
      hash
      value {
        status
        executedBlock {
          block {
            chainId
            epoch
            incomingBundles {
              origin
              bundle {
                height
                timestamp
                certificateHash
                transactionIndex
                messages {
                  authenticatedSigner
                  grant
                  refundGrantTo
                  kind
                  index
                  message
                }
              }
              action
            }
            operations
            height
            timestamp
            authenticatedSigner
            previousBlockHash
          }
          outcome {
            messages {
              destination
              authenticatedSigner
              grant
              refundGrantTo
              kind
              message
            }
            stateHash
            oracleResponses
            events {
              streamId {
                applicationId
                streamName
              }
              key
              value
            }
          }
        }
      }
    }
  }
`

const BLOCK_MATERIAL = gql`
  query blockMaterial($chainId: ChainId!) {
    blockMaterial(chainId: $chainId) {
      incomingBundles {
        action
        bundle {
          height
          timestamp
          certificateHash
          transactionIndex
          messages {
            authenticatedSigner
            grant
            refundGrantTo
            kind
            index
            message
          }
        }
        origin
      }
      localTime
      round
    }
  }
`

const OPEN_CHAIN = gql`
  mutation openChain($publicKey: PublicKey!) {
    claim(publicKey: $publicKey) {
      messageId
      chainId
      certificateHash
    }
  }
`

const WALLET_INIT_WITHOUT_KEYPAIR = gql`
  mutation walletInitWithoutKeypair(
    $publicKey: PublicKey!
    $signature: Signature!
    $faucetUrl: String!
    $chainId: ChainId!
    $messageId: MessageId!
    $certificateHash: CryptoHash!
  ) {
    walletInitWithoutKeypair(
      publicKey: $publicKey
      signature: $signature
      faucetUrl: $faucetUrl
      chainId: $chainId
      messageId: $messageId
      certificateHash: $certificateHash
    )
  }
`

const EXECUTE_BLOCK_WITH_FULL_MATERIALS = gql`
  mutation executeBlockWithFullMaterials(
    $chainId: ChainId!
    $operations: [Operation!]!
    $incomingBundles: [UserIncomingBundle!]!
    $localTime: Timestamp!
  ) {
    executeBlockWithFullMaterials(
      chainId: $chainId
      operations: $operations
      incomingBundles: $incomingBundles
      localTime: $localTime
    ) {
      executedBlock {
        block {
          chainId
          epoch
          height
          timestamp
          authenticatedSigner
          previousBlockHash
          incomingBundles {
            origin
            bundle {
              height
              timestamp
              certificateHash
              transactionIndex
              messages {
                authenticatedSigner
                grant
                refundGrantTo
                kind
                index
                message
              }
            }
            action
          }
          operations
        }
        outcome {
          messages {
            destination
            authenticatedSigner
            grant
            refundGrantTo
            kind
            message
          }
          stateHash
          oracleResponses
          events {
            streamId {
              applicationId
              streamName
            }
            key
            value
          }
        }
      }
      validatedBlockCertificateHash
      retry
    }
  }
`

const chainId = ref('a393137daba303e8b561cb3a5bff50efba1fb7f24950db28f1844b7ac2c1cf27')
const nodeServiceUrl = ref('http://172.16.31.73:30080')
const faucetUrl = ref('http://172.16.31.73:40080')
const publicKey = ref('03a2c84b5ff2d3d1d928badc0a0f6224f423275958b5ba61964bb3fff6a834b3')
const privateKey = ref('')
const messageId = ref('')
const certificateHash = ref('')
const blockMaterial = ref(undefined as unknown)

const onOpenChainClick = async () => {
  const res = await axios.post(faucetUrl.value, {
    query: OPEN_CHAIN.loc?.source?.body,
    variables: {
      publicKey: publicKey.value
    }
  })

  console.log(res)

  const data = (res as unknown as Record<string, unknown>).data

  chainId.value = (((data as Record<string, unknown>).data as Record<string, unknown>).claim as Record<string, unknown>).chainId as string
  messageId.value = (((data as Record<string, unknown>).data as Record<string, unknown>).claim as Record<string, unknown>).messageId as string
  certificateHash.value = (((data as Record<string, unknown>).data as Record<string, unknown>).claim as Record<string, unknown>).certificateHash as string
}

const toHex = (bytes: Uint8Array) =>
  bytes.reduce((str, byte) => str + byte.toString(16).padStart(2, '0'), '')

const toBytes = (hex: string) => {
  if (hex.length % 2 !== 0) {
    throw Error('Must have an even number of hex digits to convert to bytes')
  }
  const numBytes = hex.length / 2
  const bytes = new Uint8Array(numBytes)
  for (let i = 0; i < numBytes; i++) {
    bytes[i] = parseInt(hex.substring(i * 2, (i + 1) * 2), 16)
  }
  return bytes
}

const onWalletInitWithPublicKeyClick = async () => {
  const typeNameBytes = new TextEncoder().encode('Nonce::')
  const bytes = new Uint8Array([...typeNameBytes, ...toBytes(certificateHash.value)])
  const keyPair = Ed25519SigningKey.from_bytes(new Memory(toBytes(privateKey.value)))
  const signature = toHex(keyPair.sign(new Memory(bytes)).to_bytes().bytes)

  const res = await axios.post(nodeServiceUrl.value, {
    query: WALLET_INIT_WITHOUT_KEYPAIR.loc?.source?.body,
    variables: {
      publicKey: publicKey.value,
      signature,
      faucetUrl: faucetUrl.value,
      chainId: chainId.value,
      messageId: messageId.value,
      certificateHash: certificateHash.value
    }
  })

  console.log(res)
}

const onGetBlockClick = async () => {
  const res = await axios.post(nodeServiceUrl.value, {
    query: BLOCK.loc?.source?.body,
    variables: {
      chainId: chainId.value
    }
  })
  console.log(res)
}

const onGetBlockMaterialClick = async () => {
  const res = await axios.post(nodeServiceUrl.value, {
    query: BLOCK_MATERIAL.loc?.source?.body,
    variables: {
      chainId: chainId.value
    }
  })
  console.log(res)

  const data = (res as unknown as Record<string, unknown>).data

  blockMaterial.value = ((data as Record<string, unknown>).data as Record<string, unknown>).blockMaterial
}

const onExecuteBlockClick = async () => {
  const res = await axios.post(nodeServiceUrl.value, {
    query: EXECUTE_BLOCK_WITH_FULL_MATERIALS.loc?.source?.body,
    variables: {
      chainId: chainId.value,
      operations: [],
      incomingBundles: (blockMaterial.value as Record<string, unknown>).incomingBundles,
      localTime: (blockMaterial.value as Record<string, number>).localTime
    }
  })
  console.log(res)
}

onMounted(async () => {
  await Berith.initBundledOnce()
  privateKey.value = toHex((Ed25519SigningKey.random().to_bytes().bytes))
  const keyPair = Ed25519SigningKey.from_bytes(new Memory(toBytes(privateKey.value)))
  publicKey.value = toHex(keyPair.public().to_bytes().bytes)
})

</script>
