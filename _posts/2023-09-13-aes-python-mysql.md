---
title: Python과 MySQL 간 AES 암호화
date: 2023-09-13 20:20:20 +0900
categories: [encryption]
tags: [aes, python, mysql]    # TAG names should always be lowercase
---

## 상황
- MySQL DB 테이블의 특정 컬럼 데이터를 AES로 암호화하고자 한다.  
- 암호화 되지 않은 데이터가 이미 DB에 있기 때문에  
  DB의 aes_encrypt 명령어로 일괄 암호화 한다.  
- Python 프로그램에서도 aes.encrpt, aes.decrypt 함수를 구현하고  
  DB 일괄 암호화 이후에는 암호화 및 복호화를 Python에서 진행한다.  

## 중요한 점
- DB에서 AES로 암호화한 메세지 hash를  
  Python에서 복호화할 수 있어야 한다.  
- 이를 위해 AES 암호화 시 사용하는 여러 값들을   
  DB와 python에서 동일하게 맞춰야 한다.  

## 환경
- Python  
    - 버전: 3.10  
    - 암호화 패키지: [pycryptodome](https://pycryptodome.readthedocs.io/en/latest/src/introduction.html){:target="_blank"}  
- MySQL  
    - 버전: 5.7 이상  

## AES
- AES 상세 설명  
    - [[JAVA] 자바 AES 암호화 하기 (AES-128, AES-192, AES-256)](https://veneas.tistory.com/entry/JAVA-%EC%9E%90%EB%B0%94-AES-%EC%95%94%ED%98%B8%ED%99%94-%ED%95%98%EA%B8%B0-AES-128-AES-192-AES-256){:target="_blank"}  
- 용어 정리  
    - 메세지  
        - 암호화 할 대상  
    - 키  
        - 암호화에 사용하는 키  
          (대칭키 암호화 방식이니 키가 하나다)  
    - IV(Initial Vector)  
        - CBC(cipher-block chaning) 모드에서  
          최초 블록을 암호화할 때 사용하는 값이다.  
    - 설명  
        - 대칭키 암호화 방식으로 키 하나로 암호화 한다.  
        - AES는 128bit(16byte) 블록 단위로 암호화한다.  
        - 키는 128bit, 192bit(24byte), 256bit(32byte)의 크기를 가질 수 있다.  
        - IV는 블록크기와 같이 128bit이어야 한다.  
- 운용방식  
    - AES는 메세지를 블록 단위로 쪼개어 암호화한다.  
    - 이때 암호화 방식에 따라 다른 운용방식을 갖는다.  
    - DB에서 대표적으로 제공하는 모드는 ECB, CBC이다.  
    - ECB  
        - 쪼개진 메세지 블록을 암호화할 때 최초 입력한 키로   
          암호화한다.  
        - 여러 블록을 암호화 할 때 동일한 키를 쓰기 때문에  
          한 블록이라도 키가 노출되면 모든 블록이 노출된다는 단점이 있다.  
        - [ecb 설명 및 취약점](https://lactea.kr/entry/ECB-%EC%84%A4%EB%AA%85-%EB%B0%8F-%EC%B7%A8%EC%95%BD%EC%A0%90){:target="_blank"}  
        - 위 글에 따르면 암호화 결과값이 같은 블록이 여러개 생길 수 있고  
          이는 해커가 해킹 시작점을 찾기 쉽게 하여  
          무차별 대입 공격의 경우의 수를 줄일 수 있는 것으로 추정된다.  
    - CBC  
        - 쪼개진 메세지 블록을 순차적으로 나열하고  
          이전 블록의 암호화 결과와 현재 블록 값을 XOR 연산한 뒤  
          키를 이용하여 암호화하는 방식이다.  
        - 최초 블록에서는 XOR할 블록이 없다.  
          그래서 최초 블록은 IV와 XOR 연산을 한다.  
        - CBC는 ECB보다 보안성이 높고  
          Python와 DB 모두 지원하기에 CBC 모드를 사용한다.  
- 패딩(padding)  
    - 메세지는 블록(여기선 16byte) 단위 크기를 가져야 한다(n * 16byte).  
      메세지의 끝 부분이 블록크기보다 작다면 패딩을 해야한다.  
    - 패딩은 Python PyCryptodome과 MySQL 모두 PKCS#7이 패딩 기본값이다.  

## MySQL DB
```  
-- 모드 설정  
SET block_encryption_mode = 'aes-256-cbc';  
        
-- 인자  
set @message = 'my message';  
set @key = UNHEX(sha2('hello world', 256));  
set @iv = CONVERT ('1234567890123456' USING UTF8);  
        
-- 암호화  
select HEX(AES_ENCRYPT(CONVERT(@message USING UTF8), @key, @iv)) AS `RESULT`;  
        
-- 복호화  
select CONVERT(AES_DECRYPT(UNHEX('1362343BD1B9633175CF6FBEFCF96BE3'), @key, @iv) USING UTF8) AS `RESULT`;  
```  

## python
```  
from Crypto.Cipher import AES  
from Crypto.Util.Padding import pad, unpad  
import hashlib  
        
key_phrase = 'hello world  
iv = '1234567890123456'  
        
def encrypt(message):  
  global key_phrase  
  global iv  
  key = hashlib.sha256(key_phrase .encode('UTF-8')).digest()  
  # iv is must be 16 bytes after encoding utf8  
  converted_iv = iv.encode('UTF-8')  
  assert len(converted_iv) == 16  
  cipher = AES.new(key, AES.MODE_CBC, converted_iv)  
            
  padded_msg = pad(message.encode('UTF-8'), 16, 'pkcs7')  
  cipher_text = cipher.encrypt(padded_msg)  
            
  return cipher_text.hex()  
        
def decrypt(hex_str):  
  global key_phrase  
  global iv  
  key = hashlib.sha256(key_phrase .encode('UTF-8')).digest()  
  # iv is must be 16 bytes after encoding utf8  
  converted_iv = iv.encode('UTF-8')  
  assert len(converted_iv) == 16  
  cipher = AES.new(key, AES.MODE_CBC, converted_iv)  
            
  converted = bytes.fromhex(hex_str)  
  padded_msg = cipher.decrypt(converted)  
  plain_text = unpad(padded_msg, 16, 'pkcs7')  
            
  return plain_text.decode('UTF-8')  
        
```  

## 키와 iv
- 개요  
    - DB와 Python이 동일한 값을 가져야 한다.  
    - 키와 iv는 byte 타입이어야한다.  
    - Python, DB 모두에서 지원하는 UTF8로 문자열을 encoding하여  
      byte 타입으로 변환하여 사용한다.  
- 키  
    - 키는 256bit 길이를 쉽게 맞추기 위해  
      원하는 key_phrase 로 sha256 hash를 만들어 키로 사용한다.  
    - sha256 해시는 DB에서도 함수로 지원하므로  
      DB와 Python 모두 사용할 수 있다.  
- iv  
    - iv는 128 bit이어야한다.  
    - 임의의 128 bit 문자열을 UTF8 encoding 하여 사용한다.  

## encrypt 함수
- 인자  
    - 암호화할 메세지  
- 처리  
    - 입력 받은 메세지를 UTF8로 인코딩하여 byte 타입으로 형변환한다.  
    - 메세지를 pad 함수에 넣고 실행하여 16byte 단위로 떨어지게 패딩한다.  
    - 키와 iv를 이용하여 AES_CBC 모드 AES.new 객체를 생성한다.  
    - AES.new 객체의 encrypt 함수에 메세지를 넣고 실행하여  
      암호화된 byte 타입 결과값을 받는다.  
    - 결과값을 hex로 변환하여 리턴한다.  
      (byte 타입 결과값을 저장소에 그냥 넣을 경우  
      사람이 보기 글씨가 깨져보이기에  
      HEX로 타입 변환하여 가독성 확보한다.)  

## decrypt 함수
- 인자  
    - 암호화된 hex 문자열(이하 암호문)  
- 처리  
    - hex 암호 문자열을 byte 형으로 타입 변환한다.  
    - 키와 iv를 이용하여 AES_CBC 모드 AES.new 객체를 생성한다.  
    - AES.new 객체의 decrypt 함수에 암호문을 넣고 실행하여  
      복호화된 byte 타입 결과값을 받는다.  
    - 복호화된 암호문을 unpad 함수에 넣어 패딩을 제거한다.  
    - byte 타입 암호문을 UTF8로 decoding하여  
      사람이 읽을 수 있는 문자열로 타입 변환한다.  

## 주의사항
- Python 코드의 Crypto 라이브러리에서   
  AES 키를 16byte로 변경하면 AES-128 모드가 된다.   
- DB에서는 AES 키가 16byte라도   
  block_encryption_mode을 aes-256-cbc으로 설정하면   
  AES-128모드로 처리되지 않으니 주의해야한다.  
  (패딩을 하는 것인지 Python AES-128 모드와 값이 다르다)  
    - DB의 block_encryption_mode 기본값은 aes-128-ecb.  
- DB의 block_encryption_mode가 aes-128-ecb일 경우   
  암호화 시 iv값을 넣어도 리턴되는 HEX값이 변하지 않는다.  
- 이를 이용하여 DB모드를 간접적으로 확인할 수 있다.  

## 참고
- https://pycryptodome.readthedocs.io/en/latest/src/cipher/aes.html  
- https://dev.mysql.com/doc/refman/5.7/en/encryption-functions.html#function_aes-encrypt  
- https://extendsclass.com/mysql-online.html#  
- https://pycryptodome.readthedocs.io/en/latest/src/util/util.html  
