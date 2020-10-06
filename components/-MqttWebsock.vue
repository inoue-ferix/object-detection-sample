<template>
  <div></div>
</template>
<script>
import { watch, onMounted, defineComponent } from '@vue/composition-api'
import CryptoJS from 'crypto-js'
import moment from 'moment'
import Paho from 'paho-mqtt'

function SigV4Utils() {}

SigV4Utils.sign = function (key, msg) {
  const hash = CryptoJS.HmacSHA256(msg, key)
  return hash.toString(CryptoJS.enc.Hex)
}

SigV4Utils.sha256 = function (msg) {
  const hash = CryptoJS.SHA256(msg)
  return hash.toString(CryptoJS.enc.Hex)
}

SigV4Utils.getSignatureKey = function (
  key,
  dateStamp,
  regionName,
  serviceName
) {
  const kDate = CryptoJS.HmacSHA256(dateStamp, 'AWS4' + key)
  const kRegion = CryptoJS.HmacSHA256(regionName, kDate)
  const kService = CryptoJS.HmacSHA256(serviceName, kRegion)
  const kSigning = CryptoJS.HmacSHA256('aws4_request', kService)
  return kSigning
}
const AWS = require('aws-sdk')
const endpoint = process.env.END_POINT
const poolId = process.env.POOL_ID
const region = process.env.REGION

function getCredentials(done) {
  AWS.config.region = region
  const cognitoidentity = new AWS.CognitoIdentity()
  const params = {
    IdentityPoolId: poolId,
  }
  cognitoidentity.getId(params, function (err, objectHavingIdentityId) {
    if (err) return done(err)
    cognitoidentity.getCredentialsForIdentity(objectHavingIdentityId, function (
      err,
      data
    ) {
      if (err) return done(err)
      const credentials = {
        accessKey: data.Credentials.AccessKeyId,
        secretKey: data.Credentials.SecretKey,
        sessionToken: data.Credentials.SessionToken,
      }
      console.log(
        'CognitoIdentity has provided temporary credentials successfully.'
      )
      done(null, credentials)
    })
  })
}
function getEndpoint(regionName, awsIotEndpoint, done) {
  getCredentials(function (err, creds) {
    if (err) return done(err)

    const time = moment.utc()
    const dateStamp = time.format('YYYYMMDD')
    const amzdate = dateStamp + 'T' + time.format('HHmmss') + 'Z'
    const service = 'iotdevicegateway'
    const region = regionName
    const secretKey = creds.secretKey
    const accessKey = creds.accessKey
    const algorithm = 'AWS4-HMAC-SHA256'
    const method = 'GET'
    const canonicalUri = '/mqtt'
    const host = awsIotEndpoint
    const sessionToken = creds.sessionToken

    const credentialScope =
      dateStamp + '/' + region + '/' + service + '/' + 'aws4_request'
    let canonicalQuerystring = 'X-Amz-Algorithm=AWS4-HMAC-SHA256'
    canonicalQuerystring +=
      '&X-Amz-Credential=' +
      encodeURIComponent(accessKey + '/' + credentialScope)
    canonicalQuerystring += '&X-Amz-Date=' + amzdate
    canonicalQuerystring += '&X-Amz-SignedHeaders=host'

    const canonicalHeaders = 'host:' + host + '\n'
    const payloadHash = SigV4Utils.sha256('')
    const canonicalRequest =
      method +
      '\n' +
      canonicalUri +
      '\n' +
      canonicalQuerystring +
      '\n' +
      canonicalHeaders +
      '\nhost\n' +
      payloadHash

    const stringToSign =
      algorithm +
      '\n' +
      amzdate +
      '\n' +
      credentialScope +
      '\n' +
      SigV4Utils.sha256(canonicalRequest)
    const signingKey = SigV4Utils.getSignatureKey(
      secretKey,
      dateStamp,
      region,
      service
    )
    const signature = SigV4Utils.sign(signingKey, stringToSign)

    canonicalQuerystring += '&X-Amz-Signature=' + signature

    let wssEndpoint =
      'wss://' + host + canonicalUri + '?' + canonicalQuerystring
    wssEndpoint += '&X-Amz-Security-Token=' + encodeURIComponent(sessionToken)
    done(null, wssEndpoint)
  })
}
// このファイルだけJavaScriptです
export default defineComponent({
  name: '',
  props: {
    param: {
      type: String,
      required: false,
      default: '',
    },
  },
  setup(props) {
    const topic = 'Metadata_MQTT'
    let client
    const subscribe = () => {
      client.subscribe(topic)
      console.log('subscribed')
    }
    function send(content) {
      const message = new Paho.Message(content)
      message.destinationName = topic
      client.send(message)
      console.log('sent')
    }
    function onMessage(message) {
      console.log('message received: ' + message.payloadString)
    }
    watch(
      () => props.param,
      () => {
        send(props.param)
      }
    )

    onMounted(() => {
      intialize()
    })
    const intialize = () => {
      getEndpoint(region, endpoint, function (err, endpoint) {
        if (err) {
          console.log('failed', err)
          return
        }
        const clientId = Math.random().toString(36).substring(7)
        client = new Paho.Client(endpoint, clientId)
        const connectOptions = {
          useSSL: true,
          timeout: 3,
          mqttVersion: 4,
          onSuccess: subscribe,
        }
        client.connect(connectOptions)
        client.onMessageArrived = onMessage
        client.onConnectionLost = function (e) {
          console.log('connect lost')
          console.log(e)
          intialize()
        }
      })
    }
    return {}
  },
})
</script>
