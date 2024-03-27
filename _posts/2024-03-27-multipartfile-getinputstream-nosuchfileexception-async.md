---
title: MultipartFile.getInputStream() java.nio.file.NoSuchFileException in async
date: 2024-03-27 17:06:26 +0900
categories: [spring]
tags: [spring]    # TAG names should always be lowercase
---

## 개요
- 비동기 함수 안에서 MultipartFile.getInputStream() 실행 시  
  임시 파일이 없다는 에러가 난다.  

## 원인
- 컨트롤러에서 multipart/form-data 요청을 받으면  
  해당 파일이 임시 경로에 저장되는 듯 하다.  
- 이때 MultipartFile.getInputStream() 실행하면   
  임시 파일에서 stream 연결을 불러오는 것 같다.  
- 임시 파일은 해당 요청에 대한 응답을 해버리면 즉시 삭제하는 것 같다.  
- 떄문에 응답 후에 비동기 함수에서   
  MultipartFile.getInputStream()를 실행하면   
  해당 파일이 없다는 에러가 나는 것이다[[참고1](https://stackoverflow.com/questions/77620046/why-do-i-get-nosuchfileexception-when-using-multipartfile-async-in-java-spring){:target="_blank"}][[참고2](https://stackoverflow.com/questions/30738574/filenotfoundexception-while-uploading-multi-part-file-spring-boot){:target="_blank"}].  

## 해결방법 - 파일 byte로 변환하고 비동기 함수로 전달
- 설명  
    - 파일을 byte 형태로 변환한 뒤 비동기 함수에 넘겨주면  
      임시 파일이 삭제되는 것과 상관없이 처리가 가능하다.  
- 예시  
    - image/ImageDto.java  
      ```java  
      @Getter  
      @Setter  
      public class ImageDto {  
      	byte[] imageBytes;  
      	String originalFilename;  
      }  
      ```  
    - image/ImageController.java  
      ```java  
      @RequiredArgsConstructor  
      @RestController  
      public class ImageController {  
          @PostMapping(value = "/image")  
          public String create(@RequestParam(value = "image") @NotBlank MultipartFile[] imageFiles) throws IOException {  
                
              List<ImageDto> imageDtoList = new ArrayList<>();  
              for (MultipartFile imageFile : imageFiles) {  
                  ImageDto imageDto = new ImageDto();  
                  imageDto.setImageBytes(imageFile.getBytes());  
                  imageDto.setOriginalFilename(imageFile.getOriginalFilename());  
                
                  imageDtoList.add(imageDto);  
              }  
                
              this.saveImage(imageDtoList);  
                
              return "success";  
          }  
                
          public void saveImage(List<ImageDto> imageDtoList) {  
              CompletableFuture.runAsync(() -> {  
                  System.out.println("inner async start");  
                  try {  
                      // 이미지 저장 로직  
                      ...  
                
                  } catch (Exception e) {  
                      throw new RuntimeException(e);  
                  }  
                  System.out.println("inner async end");  
              }).exceptionally((e) -> {  
                  // 이미지 저장 실패 시 에러 핸들링  
                
                  return null;  
              });  
          }  
      }  
      ```  
- 단점  
    - 파일의 byte 형태 변환은 메모리에서 이뤄지므로  
      엄청 큰 파일을 처리하는 경우에는 사용하기 어렵다.  
    - 이 경우에는 비동기 함수 실행 전에  
      다른 스토리지에 파일을 저장해서 처리할 것 같다.  

## 참고
- [참고 - Why do I get NoSuchFileException when using multipartFile @Async in java spring boot?](https://stackoverflow.com/questions/77620046/why-do-i-get-nosuchfileexception-when-using-multipartfile-async-in-java-spring){:target="_blank"}  
- [참고 - FileNotFoundException while uploading multi part file - Spring boot](https://stackoverflow.com/questions/30738574/filenotfoundexception-while-uploading-multi-part-file-spring-boot){:target="_blank"}  
