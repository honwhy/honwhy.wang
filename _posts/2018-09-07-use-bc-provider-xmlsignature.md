---
layout: post
category: java
comments: true
title: Use Bouncy Castle Provider implementation to do RSA for XMLSignature
tags: XMLSignature
---

* content
{:toc}

## Abstract

There is an article shows demo code for making XMLSignature by using `Java XML Digital Signature API`, where it actually uses `org.jcp.xml.dsig.internal.dom.XMLDSigRI` to do DOM formation, and the first provider in the `java.security` lookup order that will support `SHA1` digestion, `SHA1withRSA` signing to do algorithm jobs.


## DEBUG the code

Firstly, just copy the code from the article,

```java
// Create a DOM XMLSignatureFactory that will be used to
// generate the enveloped signature.
XMLSignatureFactory fac = XMLSignatureFactory.getInstance("DOM");

// Create a Reference to the enveloped document (in this case,
// you are signing the whole document, so a URI of "" signifies
// that, and also specify the SHA1 digest algorithm and
// the ENVELOPED Transform.
Reference ref = fac.newReference
 ("", fac.newDigestMethod(DigestMethod.SHA1, null),
  Collections.singletonList
   (fac.newTransform
    (Transform.ENVELOPED, (TransformParameterSpec) null)),
     null, null);

// Create the SignedInfo.
SignedInfo si = fac.newSignedInfo
 (fac.newCanonicalizationMethod
  (CanonicalizationMethod.INCLUSIVE,
   (C14NMethodParameterSpec) null),
    fac.newSignatureMethod(SignatureMethod.RSA_SHA1, null),
     Collections.singletonList(ref));

// Load the KeyStore and get the signing key and certificate.
KeyStore ks = KeyStore.getInstance("JKS");
ks.load(new FileInputStream("mykeystore.jks"), "changeit".toCharArray());
KeyStore.PrivateKeyEntry keyEntry =
    (KeyStore.PrivateKeyEntry) ks.getEntry
        ("mykey", new KeyStore.PasswordProtection("changeit".toCharArray()));
X509Certificate cert = (X509Certificate) keyEntry.getCertificate();

// Create the KeyInfo containing the X509Data.
KeyInfoFactory kif = fac.getKeyInfoFactory();
List x509Content = new ArrayList();
x509Content.add(cert.getSubjectX500Principal().getName());
x509Content.add(cert);
X509Data xd = kif.newX509Data(x509Content);
KeyInfo ki = kif.newKeyInfo(Collections.singletonList(xd));

// Instantiate the document to be signed.
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setNamespaceAware(true);
Document doc = dbf.newDocumentBuilder().parse
    (new FileInputStream("purchaseOrder.xml"));

// Create a DOMSignContext and specify the RSA PrivateKey and
// location of the resulting XMLSignature's parent element.
DOMSignContext dsc = new DOMSignContext
    (keyEntry.getPrivateKey(), doc.getDocumentElement());

// Create the XMLSignature, but don't sign it yet.
XMLSignature signature = fac.newXMLSignature(si, ki);

// Marshal, generate, and sign the enveloped signature.
signature.sign(dsc);


// Output the resulting document.
OutputStream os = new FileOutputStream("signedPurchaseOrder.xml");
TransformerFactory tf = TransformerFactory.newInstance();
Transformer trans = tf.newTransformer();
trans.transform(new DOMSource(doc), new StreamResult(os));
```
follow the code comments, it is clearly several parts to making the job done, the key line is `signature.sign(dsc);`;

Secondly, debuging into the `sign` method, you will notice such as,
* `digestReference`, `digest` invocations
that is for Message Digest
* `sign` invocation
that is for Signature signing.

If you want to supply another provider to do the signing, you will have to implements the interface `SignatureMethod`, or extends the abstract class `DOMSignatureMethod`. However, the `sign` method in `DOMSignatureMethod` as you will override, is a little more complicated, even coupled with `org.jcp.xml.dsig.internal.dom.SignatureProvider`, and must take care of DOM structure yourself.

![code](https://image-static.segmentfault.com/222/171/2221712014-5b929730d10f8_articlex)


## use apache xmlsec

The apache xmlsec has supply boilerplate code to do XMLSignature, but in a more clearly style,

```java
public String signWithKeyPair(String sourceXml, KeyPair kp) throws Exception {
    PrivateKey privateKey = kp.getPrivate();
    Document doc = null;
    try (InputStream is = new ByteArrayInputStream(sourceXml.getBytes(Charset.forName("utf-8")))) {
        doc = MyXMLUtils.read(is, false);
    }

    Element root = doc.getDocumentElement();

    Element canonElem =
            XMLUtils.createElementInSignatureSpace(doc, Constants._TAG_CANONICALIZATIONMETHOD);
    canonElem.setAttributeNS(
            null, Constants._ATT_ALGORITHM, Canonicalizer.ALGO_ID_C14N_OMIT_COMMENTS
    );

    SignatureAlgorithm signatureAlgorithm =
            new SignatureAlgorithm(doc, XMLSignature.ALGO_ID_SIGNATURE_RSA_SHA1);
    XMLSignature sig =
            new XMLSignature(doc, null, signatureAlgorithm.getElement(), canonElem);

    root.appendChild(sig.getElement());
    Transforms transforms = new Transforms(doc);
    transforms.addTransform(Transforms.TRANSFORM_ENVELOPED_SIGNATURE);
    sig.addDocument("", transforms, Constants.ALGO_ID_DIGEST_SHA1);

    sig.addKeyInfo(kp.getPublic());
    sig.sign(privateKey);

    ByteArrayOutputStream bos = new ByteArrayOutputStream();

    XMLUtils.outputDOMc14nWithComments(doc, bos);
    return new String(bos.toByteArray(), "utf-8");
}
```

If use `xmlsec` and want to supply your own Provider to make signature, it will be less effort to hook into the *Architecture*. and also making more sense when you extends `SignatureAlgorithm` class, supply your *engine* to override the following methods,

```java
    @Override
    public void update(byte[] input) throws XMLSignatureException {
        try {
            engine.update(input);
        } catch (SignatureException e) {
            throw new XMLSignatureException(e);
        }
    }

    @Override
    public void update(byte input) throws XMLSignatureException {
        try {
            engine.update(input);
        } catch (SignatureException e) {
            throw new XMLSignatureException(e);
        }
    }

    @Override
    public void update(byte[] buf, int offset, int len) throws XMLSignatureException {
        try {
            engine.update(buf, offset, len);
        } catch (SignatureException e) {
            throw new XMLSignatureException(e);
        }
    }

    @Override
    public void initSign(Key signingKey) throws XMLSignatureException {
        try {
            engine.initSign((PrivateKey) signingKey);
        } catch (InvalidKeyException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void initSign(Key signingKey, SecureRandom secureRandom) throws XMLSignatureException {
        try {
            engine.initSign((PrivateKey) signingKey, secureRandom);
        } catch (InvalidKeyException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void initSign(Key signingKey, AlgorithmParameterSpec algorithmParameterSpec) throws XMLSignatureException {
        throw new XMLSignatureException("unsupported operation");
    }

    @Override
    public byte[] sign() throws XMLSignatureException {
        try {
            return engine.sign();
        } catch (SignatureException e) {
            throw new XMLSignatureException(e);
        }
    }

    @Override
    public void initVerify(Key verificationKey) throws XMLSignatureException {
        try {
            engine.initVerify((PublicKey) verificationKey);
        } catch (InvalidKeyException e) {
            throw new XMLSignatureException(e);
        }
    }

    @Override
    public boolean verify(byte[] signature) throws XMLSignatureException {
        try {
            return engine.verify(signature);
        } catch (SignatureException e) {
            throw new XMLSignatureException(e);
        }

    }
```
not need to take care of the DOM structure any more.

## use Bouncy Castle Provider

Making this post more specific, I will show more codes about extends `SignatureAlgorithm` class along with using the engine from Bouncy Castle Provider.

```java
public class BcSignatureAlgorithm extends AbstractSignatureAlgorithm {

    protected Signature engine;
    protected static BouncyCastleProvider provider = new BouncyCastleProvider();
    public BcSignatureAlgorithm(Document doc, String algorithmURI) throws XMLSecurityException {
        super(doc, algorithmURI);
        initEngine();
    }

    public BcSignatureAlgorithm(Document doc, String algorithmURI, int hmacOutputLength) throws XMLSecurityException {
        super(doc, algorithmURI, hmacOutputLength);
        initEngine();
    }

    public BcSignatureAlgorithm(Element element, String baseURI) throws XMLSecurityException {
        super(element, baseURI);
        initEngine();
    }

    public BcSignatureAlgorithm(Element element, String baseURI, boolean secureValidation) throws XMLSecurityException {
        super(element, baseURI, secureValidation);
        initEngine();
    }

    protected void initEngine() throws XMLSecurityException {
        try {
            engine = Signature.getInstance("SHA1withRSA", provider);
        } catch (NoSuchAlgorithmException e) {
            throw new XMLSignatureException(e);
        }
    }
}
```
nothing more, call the `SPI` getInstance method to get the engine object.
How to use the `BcSignatureAlgorithm`?
```java
public String signWithCert(String sourceXml, PrivateKey privateKey, X509Certificate signingCert) throws Exception {

    Document doc = null;
    try (InputStream is = new ByteArrayInputStream(sourceXml.getBytes(Charset.forName("utf-8")))) {
        doc = MyXMLUtils.read(is, false);
    }

    Element root = doc.getDocumentElement();

    Element canonElem =
            XMLUtils.createElementInSignatureSpace(doc, Constants._TAG_CANONICALIZATIONMETHOD);
    canonElem.setAttributeNS(
            null, Constants._ATT_ALGORITHM, Canonicalizer.ALGO_ID_C14N_OMIT_COMMENTS
    );

    AbstractSignatureAlgorithm signatureAlgorithm =
            new BcSignatureAlgorithm(doc, XMLSignature.ALGO_ID_SIGNATURE_RSA_SHA1);//use BcSignatureAlgorithm
    XMLSignature sig =
            new XMLSignature(doc, null, signatureAlgorithm.getElement(), canonElem);

    root.appendChild(sig.getElement());
    Transforms transforms = new Transforms(doc);
    transforms.addTransform(Transforms.TRANSFORM_ENVELOPED_SIGNATURE);
    sig.addDocument("", transforms, Constants.ALGO_ID_DIGEST_SHA1);

    //sig.addKeyInfo(signingCert);
    X509Data x509data = new X509Data(doc);
    x509data.addCertificate(signingCert);

    sig.getKeyInfo().addKeyName(signingCert.getSerialNumber().toString());
    sig.getKeyInfo().add(x509data);
    //sig.sign(privateKey);
    signatureAlgorithm.doSign(privateKey,sig.getSignedInfo());
    ByteArrayOutputStream bos = new ByteArrayOutputStream();

    XMLUtils.outputDOMc14nWithComments(doc, bos);
    return new String(bos.toByteArray(), "utf-8");
}
```
`AbstractSignatureAlgorithm` will finished the jobs that `XMLSignature.sign(PrivateKey)` used to do, actually `doSign` method copy lines from it, and make sure the signature value should set to SignatureValue Element as a text node element. Last but not least, thought `BcSignatureAlgorithm` is behind the sight, it has used the engine from the Bouncy Castle and produce the signature.

## For More Information
[Programming With the Java XML Digital Signature API](https://www.oracle.com/technetwork/articles/javase/dig-signature-api-140772.html)

[xml-sec](https://github.com/Honwhy/xml-sec)

