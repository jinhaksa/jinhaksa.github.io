---
layout: post
title: 크롤러 설계 2편 [전략패턴, 템플릿 메서드 패턴]
categories: [dev]
author: 문재범
email: jbm1871@jinhakapply.com
date: 2021-10-30
tag:
  - design pattern
  - strategy patterm
---

<!-- @format -->

# 크롤러 설계 2편 [전략패턴]

크롤러들은 동일한 API를 기반으로 동작방식이 조금씩 다른 형태였기 때문에 공통으로 동작하는 코드, 그리고 크롤러마다 변하는 코드가 존재했다.

그러나 기존의 코드는 이러한 비슷하거나 혹은 아예 동일한 코드들도 모두 구현 클래스마다 따로 정의해서 사용하고 있었다. 그러다 보니 코드의 중복이 발생했다.

전략패턴을 사용해서 위 문제를 해결할 수 있었다.

먼저 크롤러들의 작동방식은 대략적으로 다음과 같다.

1. 웹 사이트의 게시판 크롤링에 필요한 설정을 DB에서 불러온다.
2. 원하는 게시판의 모든 게시글을 불러온다.
3. 원하는 키워드, 날짜 기준에 따라서 해당하는 게시글만 추출한다.
4. 해당하는 게시물을 호출해 내용을 크롤링한다.
5. 해당하는 게시물을 호출해 내용을 캡처하고 서버에 저장한다.
6. 해당 게시물의 첨부파일 내역을 불러온다.
7. 첨부파일 내역을 서버로 다운로드한다.
8. 2~7중에 하나라도 실패하면 서버에 로그를 남긴다.
9. 7까지 완료하면 모든 결과를 DB에 저장하고 서버에 로그를 남긴다.

위의 과정에서 1,8,9번은 크롤러 별로 동작 방식이 구분되는 않아도 된다.

나머지 2,3,4,5,6,7 번은 각각의 구현 크롤러마다 어느부분은 동일하고 어느부분은 다를수 있는 부분이다.

![process2](/assets/img/posts/dev/2021-10-30-desgin-crawler-2/process2.png)

2,3,4,5,6,7번은 추후에 추가되거나 구현 방식이 바뀌어야 할 가능성이 크다. 또한 A크롤러에서 사용한 게시물 캡처 방식을 B크롤러에서도 사용할 수 있다.

그렇기 때문에 위 6가지 행동들은 특정 구현에 의존해서는 안되고 독립적으로 관리되어야 한다.

이제 위 6가지의 행동을 크롤러 클래스와 분리하고 각 행동을 구현하면 크롤러 클래스와 각 행동들은 독립적이게 된다.

크롤러 클래스의 소스코드와 별개로 얼마든지 새로운 동작을 정의하고, 같은 동작을 다른 클래스에서 사용함으로써 코드의 중복을 최소화할 수 있다.

![](/assets/img/posts/dev/2021-10-30-desgin-crawler-2/strategy.png)

행동 별로 알고리즘 군을 정의하고 각각 캡슐화해서 사용할 수 있도록 한다. 특정 행동을 사용하는 클라이언트(여기에서는 각 크롤러)와는 독립적으로 행동을 변경할 수 있다.

```typescript
// 팩토리 메서드
getScrapper(config: ScrapableMetaData) {
    const boardTypeCode = config.boardTypeCode;
    switch (boardTypeCode) {
      case 'C':
        // BaseScrapper 는 기본 동작을 갖고 있기에 그냥 반환한다.
        return new BaseScrapper();
      case 'S':
        const javaScriptScrapper = new BaseScrapper();
        javaScriptScrapper.board = new JavaScriptBoard();
        javaScriptScrapper.boardfilter = new JavaScriptBoardFilter();
        javaScriptScrapper.capture = new JavaScriptCapture();
        javaScriptScrapper.contents = new JavaScriptContents();
        javaScriptScrapper.attachment = new JavaScriptAttachment();
        return javaScriptScrapper;
      case 'FS':
        const javaScriptWithoutDownloadScrapper = new BaseScrapper();
        javaScriptWithoutDownloadScrapper.board = new JavaScriptBoard();
        javaScriptWithoutDownloadScrapper.boardfilter = new JavaScriptBoardFilter();
        javaScriptWithoutDownloadScrapper.capture = new JavaScriptCapture();
        javaScriptWithoutDownloadScrapper.contents = new JavaScriptContents();
        javaScriptWithoutDownloadScrapper.attachment = new JavaScriptAttachment();
        javaScriptWithoutDownloadScrapper.attachmentDownload = new NotDownload();
        return javaScriptWithoutDownloadScrapper;
      default:
        break;
    }
  }

```

각 게시판의 유형에 따라서 동적으로 각기 다른 동작을 하는 크롤러 인스턴스를 반환하면 클라이언트는 해당 팩토리 메서드가 반환해주는 크롤러의 Scrap() 메서드만 이용해서 클라이언트는 알맞게 게시판의 내용을 스크랩할 수 있다.

![](/assets/img/posts/dev/2021-10-30-desgin-crawler-2/final-process.png)

스크래퍼의 대략적인 코드는 아래와 같다.

```typescript
abstract class Scrapper {
  protected abstract _scrap(scrapingExcutionIdx: number): any;
  startScrap() {
    try {
      // 공통으로 동작해야하는 부분 (DB 접근, 로깅.. 등)
      const scrapResult = _scrap();
      // 공통으로 동작해야하는 부분 (DB 저장)
    } catch {
      // 로깅
    }
  }

  private _board: Board = null;
  public set board(board: Board) {
    this._board = board;
  }
  public get board() {
    return this._board;
  }
  protected async crawlBoard() {
    return await this.board.crawlBoard();
    // 소스코드와 구현 분리
  }

  // 이하 생략...
}

class commonScrapper extends Scrapper {
  async _scrap() {
    try {
      // Scrapper 마다 다른 구현
      const image = await this.captureDetail();
      const contents = await this.crawlDetail();
      const attachment = await this.getAttachmentInfo();
      const downloadFiles = await this.downloadAttachment(attachment);

      return {
        image,
        contents,
        attachment,
        downloadFiles,
      };
    } catch {
      // 로깅
    }
  }
}
```

실제 구현 부분은 생략되었지만, 위의 팩토리메서드와 함께 보면 흐름은 이해가될 것이다.

기존 팩토리 메서드를 사용했을 때 각각 구현 클래스의 코드가 상당부분 중복되는 부분을 해결하고,

앞으로 새로운 유형의 게시판이 만들어져 새로운 동작방식이 필요할 때 자유롭게 세부동작을 추가해 새로운 크롤러를 만들수 있게 되었다.

크롤러를 구현하며 어떻게 수 많은 게시판에 대응이 되는 크롤러를 만들 수 있을까? 라는 의문이 많이 들었다.

게시판의 동작 방식은 아주 많다. 그렇지만 기본적으로 게시판에서 게시글 리스트가 있는 형태는 변함이 없었다.

그리고 A게시판과 B게시판의 내용을 가져올 때는 동일한 로직으로 처리하지만,

게시글을 탐색하는 로직은 또 다를수도 있다.

따라서 언제든지 게시판 유형에 맞게 동작 방식이 변경되어야 한다.

적절한 디자인패턴을 프로그램에 적용하면 더욱 유지보수가 쉬운 프로그램을 만들 수 있다.

현재까지 대략 반정도의 게시판에서 크롤링할 수 있다. 나머지는 새로운 로직이 필요하다.

디자인패턴의 적용으로 새로운 로직을 기존 소스코드에 적용하는게 어렵지 않다.

디자인패턴을 사용하지 않았더라면 수정하는데 많은 시간이 소요됬을 것이다.
