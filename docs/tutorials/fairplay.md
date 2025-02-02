# FairPlay Support

When using native `src=` playback, we support using FairPlay on Safari.
Adding FairPlay support involves a bit more work than other key systems.


## Server certificate

All FairPlay content requires setting a server certificate.  This is set in the
Player configuration:

```js
const req = await fetch('https://example.com/cert.der');
const cert = await req.arrayBuffer();

player.configure('drm.advanced.com\\.apple\\.fps\\.1_0.serverCertificate',
                 new Uint8Array(cert));
```


## Content ID

Some FairPlay content use custom signaling for the content ID.  The content ID
is used by the browser to generate the license request.  If you don't use the
default content ID derivation, you need to specify a custom init data transform:

```js
player.configure('drm.initDataTransform', (initData, initDataType) => {
  if (initDataType != 'skd')
    return initData;

  // 'initData' is a buffer containing an 'skd://' URL as a UTF-8 string.
  const skdUri = shaka.util.StringUtils.fromBytesAutoDetect(initData);
  const contentId = getMyContentId(sdkUri);
  const cert = player.drmInfo().serverCertificate;
  return shaka.util.FairPlayUtils.initDataTransform(initData, contentId, cert);
});
```

## License wrapping

Some FairPlay servers need to accept the license request in a different format
or give the response in a different format.  For more info, see the general
{@tutorial license-wrapping} tutorial:

```js
player.getNetworkingEngine().registerRequestFilter((type, request) => {
  if (type != shaka.net.NetworkingEngine.RequestType.LICENSE) {
    return;
  }

  const originalPayload = new Uint8Array(request.body);
  const base64Payload =
      shaka.util.Uint8ArrayUtils.toStandardBase64(originalPayload);
  const params = 'spc=' + base64Payload;
  request.headers['Content-Type'] = 'application/x-www-form-urlencoded';
  request.body = shaka.util.StringUtils.toUTF8(encodeURIComponent(params));
});

player.getNetworkingEngine().registerResponseFilter((type, response) => {
  if (type != shaka.net.NetworkingEngine.RequestType.LICENSE) {
    return;
  }

  let responseText = shaka.util.StringUtils.fromUTF8(response.data);
  // Trim whitespace.
  responseText = responseText.trim();

  // Look for <ckc> wrapper and remove it.
  if (responseText.substr(0, 5) === '<ckc>' &&
      responseText.substr(-6) === '</ckc>') {
    responseText = responseText.slice(5, -6);
  }

  // Decode the base64-encoded data into the format the browser expects.
  response.data = shaka.util.Uint8ArrayUtils.fromBase64(responseText).buffer;
});
```

## DRM Engine

Some FairPlay support need to be detected in order to change source to HLS if FairPlay supported (AFAIK currently FairPlay is supported only within HLS and not publicly supported in DASH). You can query EME yourself, directly, and only query FairPlay, rather than the full list of key systems probed by `DrmEngine`. 

```js
async function isFairPlaySupported() {
  config = {
    initDataTypes: ['cenc', 'sinf', 'skd'],
    videoCapabilities: [{contentType: 'video/mp4; codecs="avc1.42E01E"'}],
  };
  try {
    await navigator.requestMediaKeySystemAccess('com.apple.fps', [config]);
    return true;
  } catch (error) {
    return false;
  }
}

await isFairPlaySupported();
```
