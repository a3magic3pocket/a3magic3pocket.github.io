---
title: Golang에서 python generator 구현하기
date: 2021-05-18 17:59:00 +0900
categories: [Golang]
tags: [goroutine, bufio, generator] # TAG names should always be lowercase
---

## 개요
- Golang으로 작업 중 엄청 큰 텍스트 파일을 읽을 일이 생겼는데  
python에서 generator로 읽어왔던 기억이 나서 비슷하게 구현하여 기록한다.

## python에서 파일 읽기
- 기본적으로 python에서 파일을 읽을 때는 fileObject.read()함수를 쓸 수 있다.
- fileObject.read() 함수는  
    fileObject가 바이너리일 경우 현재 위치에서 몇 바이트까지 읽을지를 인자로 받는다.  
    fileObject가 텍스트일 경우 현재 위치에서 몇개의 문자열까지 읽을지를 인자로 받는다.  
    인자를 입력하지 않을 경우, 전체 파일을 읽는다(이때 너무 파일이 크면 메모리 부족 에러가 발생한다).  
- 예시 텍스트 파일
    ```
    # test.txt
    first_line
    second_line
    ```
- 한 글자씩 불러오기
    ```
    with open("test.txt") as f:
        i = 1
        while True:
            # 1 문자열 읽기(바이너리 파일일 경우 1byte 읽기)
            chunk = f.read(1)
            print(chunk)

            # chunk가 빈 문자열이면 종료
            if chunk == "":
                break

            # 파일 시작 부분에서 i 문자열 뒤로 이동
            f.seek(i)
            i += 1
    ```
- 한 줄씩 불러오기
    - 보통 텍스트 파일 로드 시 개행(newline) 단위로 구분하여 불러온다.
    - python에서는 fileObject를 for loop로 돌리면 개행 별로 구분하여 한 줄 씩 리턴한다.
        ```
        with open("test.txt") as f:
            for row in f:
                print(row)
        ```

## python에서 generator
- iterator와 같이 행동하는 함수(for loop에서 하나씩 꺼내기 가능)를 generator 함수라고 한다.
- 보통 한 번에 메모리에 올리기에는 너무 큰 파일을 불러오거나,  
로깅과 같은 미들웨어를 구현할 때 많이 사용한다.
- return 대신 yield를 쓰면 알아서 매 iteration 마다 element를 하나씩 리턴한다.
- ```
    def read_file_gen():
        with open("test.txt") as f:
            for row in f:
                yield row
    
    for row in read_file_gen():
        print(row)
    ```

## golang에서 파일 읽기
- golang에서 주로 쓰는 파일 읽기 함수는 bufio.NewSwcanner()이다.
- 파일을 읽어온 후 행 구분 방법을 선택하고, 한 줄 씩 스캔한다.  
    (예시에서는 개행문자(\n)로 행을 구분한다(bufio.ScanLines))
    ```
    file, err := os.Open(filePath)
    if err != nil {
        fmt.Println("error: ", err.Error())
        return
    }

    # 여기서는 개행 별로 구분
    scanner := bufio.NewScanner(file)
    scanner.Split(bufio.ScanLines)
    for scanner.Scan() {
        fmt.Println(scanner.Text())
    }
    ```

## golang에서 generator와 비슷하게 구현하기
- goroutine과 channel을 이용하여 python generator와 비슷하게 구현한다.
- generator 역할을 하는 함수 내부에서 익명함수를 goroutine으로 실행하고 실행결과를 channel에 저장한다.
- generator 역할을 하는 함수의 return 값은 channel이며,  
main 함수에서 generator 함수를 for ... range로 호출한다.
    ```
    // ExptGen : "1","2","3","4","5"를 리턴하는 함수
    func ExptGen() (c chan string) {
        c = make(chan string, 1)
        go func() {
            defer close(c)
            testInts := []int{1, 2, 3, 4, 5}
            for i, val := range testInts {
                c <- strconv.Itoa(val)
            }
        }()
        return c
    }

	for value := range ExptGen() {
		fmt.Println("value", value)
	}
    ```
- 여러 goroutine을 돌리면서 결과를 하나의 channel로 리턴하고 싶을 때에는 sync.WaitGroup을 사용한다.
- 예시로 두 goroutine을 사용하여 하나의 channel에 결과를 저장하는 generator 함수를 구현한다.
- 이 구현에서는 두 goroutine이 동기적으로 실행되지 않기 때문에 순서가 보장되지 않는다.
    ```
    // ExptGen : 두 goutine을 사용한 generator 예시
    func ExptGen() (c chan string) {
        c = make(chan string, 1)

        var wg sync.WaitGroup
        wg.Add(1)
        go func() {
            defer wg.Done()
            testInts := []int{1, 2, 3, 4, 5}
            for i, val := range testInts {
                c <- strconv.Itoa(val)
            }
        }()
        wg.Add(1)
        go func() {
            defer wg.Done()
            testInts := []int{6, 7, 8, 9, 10}
            for i, val := range testInts {
                c <- strconv.Itoa(val)
            }
        }()

        go func() {
            wg.Wait()
            close(c)
        }()

        return c
    }

	for value := range ExptGen() {
		fmt.Println("value", value)
	}
    ```

## golang에서 generator로 파일 읽기
- golang에서 for ... range 방식 구문을 쓸때    
  range 뒤에 channel을 받을 경우, 오직 channel만 단독으로 받아야한다.
- generator 함수에서 channel, error를 같이 리턴했을 때 for ... range에서 error를 받을 수 없다.
- 때문에 channel와 error를 구조체로 묶어 이를 받는 channel을 만든 뒤 리턴하여 처리한다.
    ```
    type FileLine struct {
        Line string
        Err  error
    }

    func ReadLines(filePath string) (fileLineChan chan FileLine) {
        fileLineChan = make(chan FileLine, 1)
        go func() {
            var fileLine FileLine
            defer close(fileLineChan)

            file, err := os.Open(filePath)
            if err != nil {
                fileLine.Err = err
                fileLineChan <- fileLine
                return
            }

            scanner := bufio.NewScanner(file)
            scanner.Split(bufio.ScanLines)
            for scanner.Scan() {
                fileLine.Line = scanner.Text()
                fileLineChan <- fileLine
            }

        }()
        return fileLineChan
    }

	var i = 0
	for fileLine := range ReadLines("test.txt") {
        if fileLine.Err != nil {
            fmt.Println(fileLine.Err.Error())
            break
        }
		fmt.Println("fileLine.Line", fileLine.Line)
		fmt.Println("fileLine.Err", fileLine.Err)
	}
    ```

## 참고
- [How do I read in a large flat file](https://stackoverflow.com/questions/29313133/how-do-i-read-in-a-large-flat-file){:target="\_blank"}
- [golang: read file generator](https://stackoverflow.com/questions/37458080/golang-read-file-generator){:target="\_blank"}
- [golang - channel 사용하기](https://jacking75.github.io/go_channel_howto/){:target="\_blank"}
- [[golang] 채널](https://brownbears.tistory.com/315){:target="\_blank"}
- [Wait result of multiple goroutines](https://stackoverflow.com/questions/46906647/wait-result-of-multiple-goroutines){:target="\_blank"}
- [7. Input and Output - 7.2.1. Methods of File Objects](https://docs.python.org/3/tutorial/inputoutput.html){:target="\_blank"}
- [Python Generators(wiki)](https://wiki.python.org/moin/Generators){:target="\_blank"}
- [How to Use Generators and yield in Python](https://realpython.com/introduction-to-python-generators/){:target="\_blank"}
- [예제로 배우는 파이썬 프로그래밍](http://pythonstudy.xyz/python/article/206-%ED%8C%8C%EC%9D%BC-%EB%8D%B0%EC%9D%B4%ED%83%80-%EC%B2%98%EB%A6%AC){:target="\_blank"}
