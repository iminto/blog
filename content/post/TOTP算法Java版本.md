---
title: TOTP算法Java版本
date: 2018/01/08 20:21:52
tags: [Java]
---
#### TOTP 概念
TOTP - Time-based One-time Password Algorithm is an extension of the HMAC-based One Time Password algorithm HOTP to support a time based moving factor.

TOTP（基于时间的一次性密码算法）是支持时间作为动态因素基于HMAC一次性密码算法的扩展。它是OTP算法的一种

算法如下:
TOTP = Truncate(HMAC-SHA-1(K, (T - T0) / X))

K 共享密钥
T 时间
T0 开始计数的时间步长
X 时间步长
#### 代码实现
最简实现需要如下两个类
1.Base32.java
```java
public class Base32 {

	private static final char[] ALPHABET = { 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O',
			'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', '2', '3', '4', '5', '6', '7' };

	private static final byte[] DECODE_TABLE;

	static {
		DECODE_TABLE = new byte[128];

		for (int i = 0; i < DECODE_TABLE.length; i++) {
			DECODE_TABLE[i] = (byte) 0xFF;
		}
		for (int i = 0; i < ALPHABET.length; i++) {
			DECODE_TABLE[(int) ALPHABET[i]] = (byte) i;
			if (i < 24) {
				DECODE_TABLE[(int) Character.toLowerCase(ALPHABET[i])] = (byte) i;
			}
		}
	}

	public static String encode(byte[] data) {

		char[] chars = new char[((data.length * 8) / 5) + ((data.length % 5) != 0 ? 1 : 0)];

		for (int i = 0, j = 0, index = 0; i < chars.length; i++) {
			if (index > 3) {
				int b = data[j] & (0xFF >> index);
				index = (index + 5) % 8;
				b <<= index;
				if (j < data.length - 1) {
					b |= (data[j + 1] & 0xFF) >> (8 - index);
				}
				chars[i] = ALPHABET[b];
				j++;
			} else {
				chars[i] = ALPHABET[((data[j] >> (8 - (index + 5))) & 0x1F)];
				index = (index + 5) % 8;
				if (index == 0) {
					j++;
				}
			}
		}

		return new String(chars);
	}

	public static byte[] decode(String s) throws Exception {

		char[] stringData = s.toCharArray();
		byte[] data = new byte[(stringData.length * 5) / 8];

		for (int i = 0, j = 0, index = 0; i < stringData.length; i++) {
			int val;

			try {
				val = DECODE_TABLE[stringData[i]];
			} catch (ArrayIndexOutOfBoundsException e) {
				throw new Exception("Illegal character");
			}

			if (val == 0xFF) {
				throw new Exception("Illegal character");
			}

			if (index <= 3) {
				index = (index + 5) % 8;
				if (index == 0) {
					data[j++] |= val;
				} else {
					data[j] |= val << (8 - index);
				}
			} else {
				index = (index + 5) % 8;
				data[j++] |= (val >> index);
				if (j < data.length) {
					data[j] |= val << (8 - index);
				}
			}
		}

		return data;
	}
}
```
2.GoogleAuthenticator.java
```java
import javax.crypto.spec.SecretKeySpec;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.util.Base64;

import javax.crypto.Mac;

public class GoogleAuthenticator {

    // taken from Google pam docs - we probably don't need to mess with these
    public static final int SECRET_SIZE = 10;

    public static final String SEED = "g8GjEvTbW5oVSV7avLBdwIHqGlUYNzKFI7izOF8GwLDVKs2m0QN7vxRs2im5MDaNCWGmcD2rvcZx";

    public static final String RANDOM_NUMBER_ALGORITHM = "SHA1PRNG";

    int window_size = 3; // default 3 - max 17 (from google docs)最多可偏移的时间

    /**
     * set the windows size. This is an integer value representing the number of 30 second windows
     * we allow
     * The bigger the window, the more tolerant of clock skew we are.
     * @param s window size - must be >=1 and <=17. Other values are ignored
     */
    public void setWindowSize(int s) {
        if (s >= 1 && s <= 17)
            window_size = s;
    }

    /**
     * Generate a random secret key. This must be saved by the server and associated with the
     * users account to verify the code displayed by Google Authenticator.
     * The user must register this secret on their device.
     * @return secret key
     */
    public static String generateSecretKey() {
        SecureRandom sr = null;
        try {
            sr = SecureRandom.getInstance(RANDOM_NUMBER_ALGORITHM);
            sr.setSeed(Base64.getDecoder().decode(SEED));
            byte[] buffer = sr.generateSeed(SECRET_SIZE);
            Base32 codec = new Base32();
            byte[] bEncodedKey = codec.encode(buffer).getBytes();
            String encodedKey = new String(bEncodedKey);
            return encodedKey;
        }catch (NoSuchAlgorithmException e) {
            // should never occur... configuration error
        }
        return null;
    }

    /**
     * Return a URL that generates and displays a QR barcode. The user scans this bar code with the
     * Google Authenticator application on their smartphone to register the auth code. They can also
     * manually enter the
     * secret if desired
     * @param user user id (e.g. fflinstone)
     * @param host host or system that the code is for (e.g. myapp.com)
     * @param secret the secret that was previously generated for this user
     * @return the URL for the QR code to scan
     */
    public static String getQRBarcodeURL(String user, String host, String secret) {
        String format = "https://www.google.com/chart?chs=200x200&chld=M%%7C0&cht=qr&chl=otpauth://totp/%s@%s%%3Fsecret%%3D%s";
        return String.format(format, user, host, secret);
    }

    /**
     * Check the code entered by the user to see if it is valid
     * @param secret The users secret.
     * @param code The code displayed on the users device
     * @param t The time in msec (System.currentTimeMillis() for example)
     * @return
     * @throws Exception
     */
    public boolean check_code(String secret, long code, long timeMsec) throws Exception {
        Base32 codec = new Base32();
        byte[] decodedKey = codec.decode(secret);
        // convert unix msec time into a 30 second "window"
        // this is per the TOTP spec (see the RFC for details)
        long t = (timeMsec / 1000L) / 30L;
        // Window is used to check codes generated in the near past.
        // You can use this value to tune how far you're willing to go.
        for (int i = -window_size; i <= window_size; ++i) {
            long hash;
            try {
                hash = verify_code(decodedKey, t + i);
            }catch (Exception e) {
                // Yes, this is bad form - but
                // the exceptions thrown would be rare and a static configuration problem
                e.printStackTrace();
                throw new RuntimeException(e.getMessage());
                //return false;
            }
            if (hash == code) {
                return true;
            }
        }
        // The validation code is invalid.
        return false;
    }

    private static int verify_code(byte[] key, long t) throws NoSuchAlgorithmException, InvalidKeyException {
        byte[] data = new byte[8];
        long value = t;
        for (int i = 8; i-- > 0; value >>>= 8) {
            data[i] = (byte) value;
        }
        SecretKeySpec signKey = new SecretKeySpec(key, "HmacSHA1");
        Mac mac = Mac.getInstance("HmacSHA1");
        mac.init(signKey);
        byte[] hash = mac.doFinal(data);
        int offset = hash[20 - 1] & 0xF;
        // We're using a long because Java hasn't got unsigned int.
        long truncatedHash = 0;
        for (int i = 0; i < 4; ++i) {
            truncatedHash <<= 8;
            // We are dealing with signed bytes:
            // we just keep the first byte.
            truncatedHash |= (hash[offset + i] & 0xFF);
        }
        truncatedHash &= 0x7FFFFFFF;
        truncatedHash %= 1000000;
        return (int) truncatedHash;
    }
}
```
测试类如下:
```java
import org.junit.Test;

public class GoogleAuthTest {

    @Test
    public void genSecretTest() {
        String secret = GoogleAuthenticator.generateSecretKey();
        System.out.println("secret="+secret);
        String url = GoogleAuthenticator.getQRBarcodeURL("testuser", "testhost", secret);
        System.out.println("Please register " + url);
        System.out.println("Secret key is " + secret);
    }

    // Change this to the saved secret from the running the above test.
    static String savedSecret = "VGH25A7M54QPME5F";

    @Test
    public void authTest() throws Exception {
        // enter the code shown on device. Edit this and run it fast before the code expires!
        long code = 146841;
        long t = System.currentTimeMillis();
        GoogleAuthenticator ga = new GoogleAuthenticator();
        ga.setWindowSize(5); //should give 5 * 30 seconds of grace...
        boolean r = ga.check_code(savedSecret, code, t);
        System.out.println("Check code = " + r);
    }
}
```
#### OTP Auth协议
在实际使用中,通常把secret嵌入一段URL中并以二维码的形式发布,这个URL一般称为otpauth协议.其URL如下所示:
otpauth://totp/testuser@testhost?secret=VGH25A7M54QPME5F&algorithm=SHA1&digits=6&period=30
