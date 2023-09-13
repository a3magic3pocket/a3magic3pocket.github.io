---
title: PHP와 MySQL 간 AES 암호화
date: 2023-09-13 20:24:27 +0900
categories: [encryption]
tags: [aes, php, mysql]    # TAG names should always be lowercase
---

## php 예시
```  
class AES {  
        
	function encrypt(string $message, string $key, string $iv)   
	{  
		$encoding = 'UTF-8';  
	        $chiperAlgorithm = 'aes-256-cbc';  
        
		$utf8Key = mb_convert_encoding($key, $encoding, mb_detect_encoding($key));  
		$hashedKey = hash('sha256', $key);  
		$bytesKey= hex2bin($hashedKey);  
        
      	$utf8Message = mb_convert_encoding($message, $encoding, mb_detect_encoding($message));  
	        $utf8Iv = mb_convert_encoding($iv, $encoding, mb_detect_encoding($iv));  
        
		$encrypted = openssl_encrypt($utf8Message, $chiperAlgorithm, $bytesKey, OPENSSL_RAW_DATA, $utf8Iv);  
        
		return strtoupper(bin2hex($encrypted));  
	}  
        
	function decrypt(string $encrypted, string $key, string $iv)  
	{  
		$encoding = 'UTF-8';  
		$chiperAlgorithm = 'aes-256-cbc';  
        
		$utf8Key = mb_convert_encoding($key, $encoding, mb_detect_encoding($key));  
		$hashedKey = hash('sha256', $key);  
		$bytesKey= hex2bin($hashedKey);  
        
		$utf8Iv = mb_convert_encoding($iv, $encoding, mb_detect_encoding($iv));  
        
		return openssl_decrypt(hex2bin($encrypted), $chiperAlgorithm, $bytesKey, OPENSSL_RAW_DATA, $utf8Iv);  
	}  
}  
```  

## 주의사항
- openssl_encrypt 함수에 OPENSSL_RAW_DATA 옵션을 주지 않으면  
  base64로 인코딩되어서 암호화된 문자열가 리턴된다.  
- openssl_decrypt 함수에 OPENSSL_RAW_DATA 옵션을 주지 않으면  
  입력 받은 문자열이 base64 인코인되었다고 가정하고 처리한다.  
