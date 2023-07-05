# Spring Boot

## Requirements

- [Eclipse(2023-03)](https://www.eclipse.org/downloads/packages/release/2023-03/r/eclipse-ide-java-developers)
- OpenJDK 20.0.1
- Spring Boot 3.1.1
- [MySQL(MariaDB) - 10.11.3](https://mariadb.com/kb/en/mariadb-10-11-3-release-notes/)
- [MySQL Workbench - 8.0.33](https://dev.mysql.com/downloads/workbench/)

## 日記投稿Webアプリ作成

- Spring BootでWebアプリを作る題材として、ブラウザ上で動作する簡単な日記投稿Webアプリを作成します。日記の新規投稿だけでなく、編集、削除、一覧表示という基本的なCRUD機能を備えたものを作成します。
- また、今回はデータベースにMySQLを使用します。

    ### CRUDとは
    - C -> Create(データの作成)
    - R -> Read(データの取得)
    - U -> Update(データの更新)
    - D -> Delete(データの削除)


## 第一章 - Spring Bootプロジェクトのインポートと実行

### 1. プロジェクトのインポート

- 割愛

### 2. 初めてのSpring Bootアプリの実行と動作確認

- Spring Bootのプロジェクトのインポートが完了したらアプリを起動します。この状態で起動しても、何も表示されずエラーになりますが、ちゃんと起動するかの動作確認です。```DiaryApplication.java```というJavaファイルが自動生成されていると思いますのでこのファイルを右クリックし→実行→Spring Bootを選択すると起動します。
- デフォルトでは[http://localhost:8080/](http://localhost:8080/)でアクセスできるはずです。

## 第二章 - データベースの設定とJavaエンティティクラスの作成

### 1. データベースの作成と接続ユーザ追加

- データベースの作成
    ```
    -- diarydbの作成
    create database diarydb;
    use diarydb;

    -- Tableの作成
    create table diary(id integer auto_increment,body_text varchar(255) not null, create_datetime timestamp not null,primary key(id));

    -- データのInsert
    insert into diary (body_text,create_datetime) values ('I am very happy!',LOCALTIME());

    -- データの確認
    select * from diary;
    ```
- 以下のような結果が返ってくればOK

    | id | body_text | create_datetime |
    | --- | --- | --- |
    | 1 | I am very happy! | 2023-07-03T10:32:10 |

- 接続ユーザ追加
    ```
    -- ユーザ名:diary_user,パスワード:P@ssWordで作成する
    grant all on *.* to diary_user identified by 'P@ssWord';

    select user,host,password from mysql.user;
    ```
- 以下のような結果が返ってくればOK（省略あり
    | User | Host | Password |
    | --- | --- | --- |
    | diary_user | % | *X9DAKAP+D86A... |

### 2. ```application.properties```でデータベース接続設定

- 以下を記述します。
    ```
    # DB接続
    spring.datasource.url=jdbc:mysql://localhost/diarydb
    spring.datasource.username=diary_user
    spring.datasource.password=P@ssWord
    ```

### 3. エンティティクラスの作成

- Diaryテーブルを想定して、Javaのエンティティクラス```Diary.java```を作成します。
- ```com.example.diary.models```パッケージを作成し、その中に```Diary.java```を作成します。
    ```java
    package com.example.diary.models;

    import java.time.LocalDateTime;
    
    import jakarta.persistence.Column;
    import jakarta.persistence.Entity;
    import jakarta.persistence.GeneratedValue;
    import jakarta.persistence.GenerationType;
    import jakarta.persistence.Id;

    @Entity
    public class Diary {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private int id;

        @Column(name = "body_text", nullable = false)
        private String bodyText;

        @Column(name = "create_datetime", nullable = false)
        private LocalDateTime createDatetime;

        public Diary(){}
        
        public Diary(int id, String bodyText, LocalDateTime createDatetime) {
            this.id = id;
            this.bodyText = bodyText;
            this.createDatetime = createDatetime;
        }

        //Getter and Setter作ってね
    }
    ```
    - ```@Entity```はDiaryクラスがエンティティクラスであることを示すために必要です。この定義がないと、SQLの操作に失敗します。理由は```Spring Data JPA```にありますが、割愛します。一度このアノテーションを消して試してみるといいと思います。
    - ```@GeneratedValue```は主キーが自動採番されるため示す必要があります。```strategy = GenerationType.IDENTITY```は採番方法を示します。
    - ```@Column```を付けないとエンティティクラスのフィールド名とテーブルのフィールド名は同じになります。今回はMySQLのネーミングルール（snake_case）とJavaのネーミングルール(camelCase)が違うため明示的に紐づける必要があります。
        - ```name = "body_text"```は上記の理由です。
        - ```nullable = false```はフィールドの値をNull禁止にします。

## 第三章 - テーブルのCRUD操作をするリポジトリインターフェースを作成

### 1. リポジトリインターフェースの作成

- データベースのテーブル操作で基本となるCRUD操作するために```Spring Data JPAのListCrudRepository```を継承してリポジトリインターフェース```DiaryRepository.java```を作成します。
- ```DiaryRepository.java```
    ```java
    package com.example.diary.repositories;

    import org.springframework.data.repository.ListCrudRepository;

    import com.example.diary.models.Diary;

    public interface DiaryRepository extends ListCrudRepository<Diary,Integer> {
        //何も書かない
    }
    ```
- ```DiaryRepository.java```は、ただ単に、```ListCrudRepository```を継承しただけのインターフェースです。ただのインターフェースですが、```Spring Data JPA```ではこのインターフェースを実装したクラスを自動で作ってくれるため、プログラマはこの```DiaryRepository.java```を使ってDiaryテーブルの簡単なCRUD操作ができます。
    > *CrudRepositoryを継承しただけのDiaryRepositoryインターフェースを使用してできるCRUD操作には限界があります。例えば条件でテーブルのフィールドを指定してデータを取得したい場合はDiaryRepositoryインターフェースにメソッドを追加する必要があります。また、＠Queryアノテーションで実行したいSQLを直接指定することもできます。*

## 第四章 - コントローラクラスの作成、日記一覧データを取得

### 1. コントローラクラス```DiaryController```の作成

- ```DiaryController.java```
    ```java
    package com.example.diary.controllers;

    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;

    import com.example.diary.repositories.Diary;

    @Controller
    @RequestMapping("diary")
    public class DiaryController {
        @Autowired
        DiaryRepository diaryRepository;
    }
    ```
- コントローラクラスには```@Controller```をつけます。今回のWebアプリではフロントエンドはView(HTML)になります。この場合のコントローラクラスには```@Controller```を使用します。
    > *```@RestController```<br>
    > Spring BootでRest API(Web API)サービスを作成する場合はフロントエンドにView(HTML)はありません。その場合のコントローラクラスには```@RestController```をつけます。*
- ```@RequestMapping("diary")```<br>
```@Controller```クラスには```@RequestMapping```を付けています。```@RequestMapping```では、ルーティング情報(URLマッピング)を設定します。ここでは、```@RequestMapping```の引数に「```"diary"```」を指定指定ます。そのため```DiaryController```は以下のURLにリクエストされた時に受け付けます。
    ```
    http://localhost:8080/diary
    ```
- ```@Autowired```<br>
Springでは、```@Autowired```を付けるとプログラマがリポジトリクラスやサービスクラスをnewしなくても、オブジェクトを自動でセット（注入）してくれます。(Dependancy Injection)


### 2. 日記一覧データ取得メソッドの作成

- ```DiaryController.java```
    ```java
    package com.example.diary.controllers;

    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;

    import com.example.diary.repositories;
    /*
     *ここから下のimportを追加
     */
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;

    import java.util.List;

    import com.example.diary.models;

    @Controller
    @RequestMapping("diary")
    public class DiaryController {
        @Autowired
        DiaryRepository diaryRepository;

        //日記一覧の取得
        @GetMapping("summary")
        public String summary(Model model) {
            List<Diary> diaries = diaryRepository.findAll(); //diaryテーブルから日記の一覧情報を取得
            model.addAttribute("diaries", diaries);
            return "summary";
        }
    }
    ```

- ```@GetMapping("summary")```<br>
```summary```メソッドには```@GetMapping("summary")```をつけています。```@GetMapping```は```@RequestMapping```と同様にルーティング情報(URLマッピング)を設定しますが、HTTPのGETリクエストだけを受け付け、POSTリクエストは受け付けません。```@RequestMapping```は、HTTPリクエストのGET・POST関係なくURLが合っていれば受け付けます。<br>
よって```summary```メソッドは、以下のURLにHTTP GETリクエストされた時に呼び出されます。
    ```
    http://localhost:8080/diary/summary
    ```

- ```model.addAttribute("diaries", diaries)```<br>
```Model```クラスの```addAttribute```メソッドを使用することで、**サーバーサイドからフロントエンドへデータの受け渡しができる**ようになります。つまり、ここでは日記一覧データが入っている変数```diaries```をフロントエンドへ返しています。こうすることでフロントエンド側で```diaries```という変数で日記一覧データを扱えるようになります。
- ```return "summary"```<br>
```summary.html```ファイルをフロントエンドに返すことを意味しています。

## 第五章 - 日記一覧データの表示、フロントエンドの開発

### 1. Thymeleafとは

- 日記一覧データを表示するためのフロントエンドの作成ですが、Spring Bootでは純粋なHTMLファイルではなくテンプレートエンジンの**Thymeleaf**を使用します。<br>
テンプレートエンジンはHTMLのひな形がベースにあり、そのHTMLのひな形にデータを合成して、最終的に純粋なHTMLファイルを出力する事ができます。<br>
今回開発しようとしている日記一覧データの表示では日記一覧データをHTML内で合成するので、その部分は**プログラムのようなコードを書く必要**があります。
    > ***Thymeleafは他にも便利な機能やいろいろな使い方があります。***<br>
    > - *変数のエスケープ*
    > - *条件分岐や繰り返し操作*
    > - *基本オブジェクトや日付・文字列・数値のユーティリティオブジェクト*
    > - *HTMLの共通化テンプレート、フラグメント化*

### 2. テンプレートファイル```summary.html```の作成と実装

- 日記一覧データを表示するためのテンプレートファイルのThymeleafを作成します。```src/main/resources/template```ディレクトリ下に```summary.html```というファイル名で新規作成します。
    ```html
    <!DOCTYPE html>
    <html xlmns:th="http://www.thymeleaf.org">
        <head>
            <meta charset="UTF-8">
            <title>[Spring Boot] Diary web applications [Thymeleaf]</title>
        </head>
        <body>
            <h1>Diary list</h1>
            <table>
                <thead>
                    <tr>
                        <th></th>
                        <th>datetime</th>
                        <th></th>
                        <th></th>
                    </tr>
                </thead>
                <tbody>
                    <tr th:each="diary : ${diaries}">
                        <td th:text="${diary.bodyText}"></td>
                        <td th:text="${diary.createDatetime}"></td>
                        <td>
                            <a th:href="@{/diary/edit(id=${diary.id})}">edit</a>
                        </td>
                        <td>
                            <form th:action="@{/diary/delete}" method="post">
                                <input type="hidden" name="id" th:value="${diary.id}"/>
                                <input type="submit" value="delete"/>
                            </form>
                        </td>
                    </tr>
                </tbody>
            </table>
            <h3>New post</h3>
            <form th:action="@{/diary/add}" method="post">
                <input type="text" name="bodyText"/>
                <input type="submit" value="submit"/>
            </form>
        </body>
    </html>
    ```

    > ***テンプレートファイルのファイル名について***<br>
    > *テンプレートファイルのファイル名は、コントローラクラスのメソッドの戻り値に合わせる必要があります。*<br>
    > *コントローラクラス```DiaryController.java```の```summary```メソッドの戻り値は```return "summary";```でしたので、テンプレートファイルのファイル名は```summary.html```にする必要があります。*
- Thymeleafのファイルは、htmlタグで下のように```th```の名前空間（ネームスペース）を宣言します。 
    ```html
    <html xmlns:th="http://www.thymeleaf.org">
    ```
    > *この名前空間がないと動作しないわけではないがお約束で書きましょう。*
- ```<table>```タグの中で日記一覧データの表示をしています。
    ```html
    <tr th:each="diary : ${diaries}">
        <td th:text="${diary.bodyText}"></td>
        <td th:text="${diary.createDatetime}"></td>
        ...
    </tr>
    ```
    コントローラクラス```DiaryController.java```の```summary```メソッドで、```Model```クラスを使って日記一覧データを```diaries```という変数に入れてフロントエンドに返しました。<br>
    ここではその```diaries```を使っています。**```diaries```には日記データが複数入っているため```th:each```で繰り返し処理をしています**<br>
    日記データを1件ずつ```diary```という変数に入れなおし、```th:text="${diary.bodyText}"```で日記の本文を表示し、```th:text="${diary.createDatetime}"```で日記の投稿日時を表示しています。
    > ```summary.html```には書く日記の編集と削除ボタンがありますが、現段階では対応する処理を実装していませんのでボタンを押してもエラーになります。

## 第六章 - 日記の削除機能の追加

### 1. コントローラクラスに日記の削除処理メソッドを追加

- 日記一覧データを取得するための処理を書いた```DiaryController.java```に日記削除を行う```delete```メソッドを追加します。
    ```java
    package com.example.diary.controllers;

    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    
    import com.example.diary.repositories;
    import com.example.diary.models;

    import java.util.List;


    @Controller
    @RequestMapping("diary")
    public class DiaryController {
        @Autowired
        DiaryRepository diaryRepository;

        //日記一覧の取得
        @GetMapping("summary")
        public String summary(Model model) {
            List<Diary> diaries = diaryRepository.findAll(); //diaryテーブルから日記の一覧情報を取得
            model.addAttribute("diaries", diaries);
            return "summary";
        }

        //指定されたidの日記を削除する
        @PostMapping("delete")
        public String delete(@RequestParam int id) {
            diaryRepository.deleteById(id);
            return "redirect:/diary/summary";
        }
    }
    ```
- ```@PostMapping("delete")```<br>
```delete```メソッドには```@PostMapping("delete")```をつけています。```@PostMapping```は```@RequestMapping```と同様にルーティング情報(URLマッピング)を設定しますが、HTTPのPOSTリクエストだけを受け付け、GETリクエストは受け付けません。```@RequestMapping```は、HTTPリクエストのGET・POST関係なくURLが合っていれば受け付けます。<br>
よって```delete```メソッドは、以下のURLにHTTP POSTリクエストされた時に呼び出されます。
    ```
    http://localhost:8080/diary/delete
    ```
- ```return "redirect:/diary/summary";```<br>
画面のリダイレクトを行っています。このコードは以下のURLにリダイレクトされます。
    ```
    http://localhost:8080/diary/summary
    ```
- ```summary.html```の削除ボタン部分を抜粋して確認してみましょう。<br>
    ```html
    <td>
        <form th:action="@{/diary/delete}" method="post">
            <input type="hidden" name="id" th:value="${diary.id}"/>
            <input type="submit" value="delete"/>
        </form>
    </td>
    ```
    ```<input type="hidden" name="id" th:value="${diary.id}"/>```で投稿された日記のid情報を紐づけています。<br>
    しかし実際にはこのidをテーブル上に表示する必要はないので```type="hidden"```を指定して非表示にしています。<br>
    また、```name="id"```と指定することでPOSTする際にパラメータとしてidという名前でリクエストをPOSTします。URLとしては以下のようなURLになります。
    ```
    http://localhost:8080/diary/delete?id=1
    ```

## 第七章 - 日記の新規登録機能を追加

### 1. コントローラクラスに日記の新規投稿処理メソッドを追加

- ```DiaryController.java```に日記の作成を行う```add```メソッドを追加します。
    ```java
    package com.example.diary.controllers;

    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    
    import com.example.diary.repositories;
    import com.example.diary.models;

    import java.util.List;


    @Controller
    @RequestMapping("diary")
    public class DiaryController {
        @Autowired
        DiaryRepository diaryRepository;

        //日記一覧の取得
        @GetMapping("summary")
        public String summary(Model model) {
            List<Diary> diaries = diaryRepository.findAll(); //diaryテーブルから日記の一覧情報を取得
            model.addAttribute("diaries", diaries);
            return "summary";
        }

        //指定されたidの日記を削除する
        @PostMapping("delete")
        public String delete(@RequestParam int id) {
            diaryRepository.deleteById(id);
            return "redirect:/diary/summary";
        }

        //日記の新規登録
        @PostMapping("add")
        public String add(@RequestParam String bodyText) {
            Diary diary = new Diary(0, bodyText, LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS)); //ChronoUnitで秒より下を切り捨て
            diaryRepository.save(diary);
            return "redirect:/diary/summary";
        }
    }
    ```
- マッピングされるURLや、パラメータ名は```delete```と一緒ですね。説明は割愛します。
- ```Diary diary = new Diary(0, bodyText, LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));```<br>
```Diary```クラスのコンストラクタには```id```,```bodyText```,```createDatetime```の情報が必要ですのでこれらを渡してインスタンスを生成します。また、投稿日時にはミリ秒が必要ないので秒より下を切り捨てています。

## 第八章 - 登録フォームのValidation機能を追加

### 1. Spring Bootでバリデーション機能を有効にする

- Spring Bootでバリデーション機能を開発する場合、バリデーションチェックのルールを作成し、機能を活用していくことになります。<br>
プロジェクト作成時に```Validation```にチェックを入れておくことで機能を追加することができます。<br>
また、忘れていた場合でも以下の依存関係を明示してあげれば使えます。
    ```mvn
    <dependancy>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependancy>
    ```
### 2. バリデーションルールを作成する

- 今回の日記投稿Webアプリのバリデーションルールを考案します。<br>
今回は登録フォームに本文を記入して登録するだけですのでチェックする対象は本文に対してです。<br>
ここでは、**3文字以上150文字以内、空文字を禁止**としておきます。

### 3. バリデーションルールの設定

- ```Diary.java```を編集してバリデーションルールを設定していきます。
    ```java
    package com.example.diary.models;

    import java.time.LocalDateTime;
    
    import jakarta.persistence.Column;
    import jakarta.persistence.Entity;
    import jakarta.persistence.GeneratedValue;
    import jakarta.persistence.GenerationType;
    import jakarta.persistence.Id;
    /*
     * 以下のimportを追加。
     */
    import javax.validation.constraints.NotNull;
    import javax.validation.constraints.Size;

    @Entity
    public class Diary {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private int id;

        @NotNull
        @Size(min = 3, max = 150)
        @Column(name = "body_text", nullable = false)
        private String bodyText;

        @Column(name = "create_datetime", nullable = false)
        private LocalDateTime createDatetime;

        public Diary(){}
        
        public Diary(int id, String bodyText, LocalDateTime createDatetime) {
            this.id = id;
            this.bodyText = bodyText;
            this.createDatetime = createDatetime;
        }

        //Getter and Setter作ってね
    }

    ```
- このバリデーションルールを設定するフィールド名と```Thymeleaf```テンプレートファイルの```name```属性を合わせる必要があります。<br>
抜粋して確認してみましょう。
    ```html
    <form th:action="@{/diary/add}" method="post">
        <input type="text" name="bodyText"/>
        <input type="submit" value="submit"/>
    </form>
    ```
- ```@NotNull```と```@Size(min = 3, max = 150)```<br>
これらのアノテーションでNullの禁止と文字列の長さ(最小3文字、最大150文字)を指定しています。
    > ***エラーメッセージの変更***<br>
    *バリデーションチェックでエラーになった場合、画面側でエラーメッセージが表示することができますが、Spring Bootではエラーメッセージの文言を指定しなくても、デフォルトで適当な文言を表示してくれます。<br>
    エラーメッセージの文言を自分で指定する場合、バリデーションチェックのアノテーションに```message```属性を付けます。*
    >   ```java
    >   @Size(min = 3, max = 150, message = "本文は3文字から150文字の間にする必要がありまっせ！")
    >   ```
### 4. コントローラクラスの変更（```add```メソッド、```summary```メソッド）

- ```DiaryController.java```
    ```java
    package com.example.diary.controllers;

    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    
    import com.example.diary.repositories;
    import com.example.diary.models;

    import java.util.List;

    /*
     * 以下のimportを追加。
     */
    import javax.validation.Valid;
    import org.springframework.validation.BindingResult;


    @Controller
    @RequestMapping("diary")
    public class DiaryController {
        @Autowired
        DiaryRepository diaryRepository;

        //日記一覧の取得
        @GetMapping("summary")
        public String summary(Model model, Diary diary) {
            List<Diary> diaries = diaryRepository.findAll(); //diaryテーブルから日記の一覧情報を取得
            model.addAttribute("diaries", diaries);
            return "summary";
        }

        //指定されたidの日記を削除する
        @PostMapping("delete")
        public String delete(@RequestParam int id) {
            diaryRepository.deleteById(id);
            return "redirect:/diary/summary";
        }

        //日記の新規登録
        @PostMapping("add")
        public String add(Model model, @Valid Diary diary, BindingResult bindingResult) {
            if(bindingResult.hasErrors()) {
                return summary(model, diary);
            }
            diary.setCreateDatetime(LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));
            diaryRepository.save(diary);
            return "redirect:/diary/summary";
        }
    }
    ```
- ```summary```メソッドの変更点は以下の点です。
    ```diff
    - public String summary(Model model) {
    + public String summary(Model model, Diary diary) {
          List<Diary> diaries = diaryRepository.findAll(); //diaryテーブルから日記の一覧情報を取得
          model.addAttribute("diaries", diaries);
          return "summary";
      }
    ```
    - バリデーションエラー時にエラーメッセージを表示するために、フロントエンド側のHTML(```summary.html```)で```diary```を使うようになったからです。<br>
    ```summary```メソッドの引数に```Diary diary```を追加すると、```summary```メソッドが呼ばれた時に```model```に```diary```オブジェクトが自動でセットされ、そのまま```model```が```diary```オブジェクトをフロント側へ返すという流れになるので、フロント側（HTML側）で```diary```オブジェクトが使えるようになります。
- ```add```メソッドの変更点は以下の点です。
    ```diff
    - public String add(@RequestParam String bodyText) {
    + public String add(Model model, @Valid Diary diary, BindingResult bindingResult) {
    +     if(bindingResult.hasErrors()) {
    +         return summary(model, diary);
    +     }
    -     Diary diary = new Diary(0, bodyText, LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS)); //ChronoUnitで秒より下を切り捨て
    +     diary.setCreateDatetime(LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));
          diaryRepository.save(diary);
          return "redirect:/diary/summary";
      }
    ```
    - 引数に```Diary```を指定する事で、新規登録フォームの本文```bodyText```の値```Diary```のフィールド```bodyText```に入るため、今まで引数に指定していた```@RequestParam String bodyText```は必要なくなるので削除しています。<br>
    また、引数の```Diary```には```@Valid```を付けています。```@Valid```を付ける事で、フォームクラス```Diary```の各フィールドのバリデーションチェック機能が有効になります。<br>
    そして、```Diary```のバリデーションチェックの結果が、もう一つの引数の```BindingResult bindingResult```に入ります。<br>
具体的に言うと、```bindingResult```にはエラーが存在したか、エラーが何個あるか、エラー内容などの情報が入ります。一度デバッグをして```bindingResult```の値を確認すると、イメージが掴みやすくなると思います。
        > ***```@Valid、@Validated```***<br>
        > *上では引数のフォームクラスに```@Valid```を付けていますが、```@Valid```は```javax.validation.Valid```で、JavaのJakarta Bean Validationです。*<br>
        > *また、```@Valid```とは別に```@Validated```もあります。```@Validated```は```org.springframework.validation.annotation.Validated```で、Springの機能です。*
    - 次に```add```メソッド内の最初のコードです
        ```java
        if(bindingResult.hasErrors()) {
            return summary(model, diary);
        }
        ```
        ```bindingResult```の```hasErrors```メソッドでバリデーションエラーが存在するかどうかを判別し、エラーがあった場合は登録処理をせずに```summary```メソッドを呼び出します。```summary```メソッドにエラー情報を渡すことでエラー情報をフロントエンドに送ることができます。
        > *この時、```add```メソッドや```summary```メソッド内では、```model```に```diary```と```bindingResult```の値が自動的にセットされています。そのため、フロント側でエラーがあればエラーメッセージを表示するようになります。*
    - 最後に```add```メソッド内の変更箇所です。
        ```java
        diary.setCreateDatetime(LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));
        ```
        引数の```@RequestParam String bodyText```を削除したので、投稿された本文のデータは、```Diary```クラスの```bodyText```から取得しています。

### 5. フロントエンドの変更

- フロントエンドの日記一覧情報と新規登録フォームがある```summary.html```を変更します。
    ```diff
    <!DOCTYPE html>
    <html xlmns:th="http://www.thymeleaf.org">
        <head>
            <meta charset="UTF-8">
            <title>[Spring Boot] Diary web applications [Thymeleaf]</title>
        </head>
        <body>
            <h1>Diary list</h1>
            <table>
                <thead>
                    <tr>
                        <th></th>
                        <th>datetime</th>
                        <th></th>
                        <th></th>
                    </tr>
                </thead>
                <tbody>
                    <tr th:each="diary : ${diaries}">
                        <td th:text="${diary.bodyText}"></td>
                        <td th:text="${diary.createDatetime}"></td>
                        <td>
                            <a th:href="@{/diary/edit(id=${diary.id})}">edit</a>
                        </td>
                        <td>
                            <form th:action="@{/diary/delete}" method="post">
                                <input type="hidden" name="id" th:value="${diary.id}"/>
                                <input type="submit" value="delete"/>
                            </form>
                        </td>
                    </tr>
                </tbody>
            </table>
            <h3>New post</h3>
            <form th:action="@{/diary/add}" method="post">
                <input type="text" name="bodyText"/>
                <input type="submit" value="submit"/>
    +           <div th:if="${#fields.hasErrors('diary.bodyText')}" th:errors="*{diary.bodyText}"></div>
            </form>
        </body>
    </html>

    ```
    - ```summary.html```の変更点は以下の1行です。
        ```html
        <div th:if="${#fields.hasErrors('diary.bodyText')}" th:errors="*{diary.bodyText}"></div>
        ```
        新規投稿フォームの本文の内容がバリデーションチェックに引っかかった場合、エラーメッセージを表示しています。<br>
```th:if="${#fields.hasErrors('diary.bodyText')}"```で、フォームクラス```diary```のフィールド```bodyText```にエラーがあったかどうかを判別し、エラーがあれば```th:errors="*{diary.bodyText}"```でエラーメッセージを表示しています。

## 第九章 - 日記の編集画面と更新機能

### 1. 編集画面```edit.html```の追加

- フロントエンドの編集画面を作成します。編集画面には一覧画面のeditから遷移するようにしています。<br>
その部分のhtmlを抜粋します。
    ```html
    <td>
        <a th:href="@{/diary/edit(id=${diary.id})}">edit</a>
    </td>
    ```
- 編集画面への遷移URLは以下の通りです。
    ```
    http://localhost:8080/diary/edit?id=2
    ```
- 編集画面```edit.html```の作成
    ```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
        <head>
            <meta charset="UTF-8">
            <title>[Spring Boot] Diary web applications [Thymeleaf] - edit</title>
        </head>
        <body>
            <h1>Edit diary</h1>
            <form th:action="@{/diary/update}" method="post">
                <input type="text" name="bodyText" th:value="${diary.bodyText}"/>
                <input type="hidden" name="id" th:value="${diary.id}"/>
                <input type="submit" value="update"/>
                <div th:if="${#fields.hasErrors('diary.bodyText')}" th:errors="*{diary.bodyText}"></div>
                <a href="/diary/summary">Return to list</a>
            </form>
        </body>
    </html>
    ```
    画面自体は編集するためのテキストボックスと更新ボタンだけなので、とてもシンプルです。<br>
編集画面のソース（```edit.html```）を見ると、画面を開いた時にテキストボックスは編集前の日記本文を表示するために```th:value="{diary.bodyText}"```という属性を付けています。<br>
```<input type="hidden"/>```に編集する日記の```id```を保持するために```th:value="${diary.id}"```という属性を付けています。<br>
この```th:value="${…}"```は、バックエンドのコントローラクラスから受け取る情報ですので、属性の値はこれから作るコントローラクラスと合わせる必要があります。<br>
また、編集する日記本文の内容がバリデーションチェックでエラーになった場合、エラーメッセージを表示するようにしています。

### 2. コントローラクラスに編集画面の呼び出しメソッドと日記の更新処理メソッドを追加

- メソッドを追加した```DiaryController.java```です。
    ```java
    package com.example.diary.controllers;

    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    
    import com.example.diary.repositories;
    import com.example.diary.models;

    import java.util.List;

    import javax.validation.Valid;
    import org.springframework.validation.BindingResult;


    @Controller
    @RequestMapping("diary")
    public class DiaryController {
        @Autowired
        DiaryRepository diaryRepository;

        //日記一覧の取得
        @GetMapping("summary")
        public String summary(Model model, Diary diary) {
            List<Diary> diaries = diaryRepository.findAll(); //diaryテーブルから日記の一覧情報を取得
            model.addAttribute("diaries", diaries);
            return "summary";
        }

        //指定されたidの日記を削除する
        @PostMapping("delete")
        public String delete(@RequestParam int id) {
            diaryRepository.deleteById(id);
            return "redirect:/diary/summary";
        }

        //日記の新規登録
        @PostMapping("add")
        public String add(Model model, @Valid Diary diary, BindingResult bindingResult) {
            if(bindingResult.hasErrors()) {
                return summary(model, diary);
            }
            diary.setCreateDatetime(LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));
            diaryRepository.save(diary);
            return "redirect:/diary/summary";
        }

        //編集画面を表示する
        @GetMapping("edit")
        public String edit(Model model, Diary diary) {
            Diary originalDiary = diaryRepository.findById(diary.getId()).get();
            diary.setBodyText(originalDiary.getBodyText());
            diary.setCreateDatetime(originalDiary.getCreateDatetime());
            model.addAttribute("diary", diary);
            return "edit";
        }

        //日記を更新する
        @PostMapping("update")
        public String update(Model model, @Valid Diary diary, BindingResult bindingResult) {
            if (bindingResult.hasErrors()) {
                return edit(model, diary);
            }
            diary.setCreateDatetime(LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));
            diaryRepository.save(diary);
            return "redirect:/diary/summary";
        }
    }
    ```
    - ```edit```メソッド内の1行目ですが、まずは編集対象の日記をDatabaseのdiaryテーブルから取得します。
        ```java
        Diary originalDiary = diaryRepository.findById(diary.getId()).get();
        ```
        ```diary```テーブルからの取得は、リポジトリインタフェース```DiaryRepository```を使用しています。```findById```メソッドは、Spring Data JPAの```ListCrudRepository```が宣言しているメソッドで、戻り値は```Optional<T>```です。
        > ***```Optional<T>```から値の実体を取得するには```get```メソッドを実行する必要があります***<br>
        *ただし、値がない場合に実行すると例外を吐くため本来は```isPresent```メソッドでデータがあることを確認してから```get```する必要があります。<br>
        今回は、必ず取れる前提で直接```get```メソッドを実行しています。*
    - 次に、```edit```メソッド内の2行目です。```Model```クラスの```addAttribute```メソッドを使用して、編集対象の日記オブジェクト```originalDiary```をフロントエンドへ返す準備をしています。
        ```java
        model.addAttribute("diary", diary);
        ```
    - 次に、日記の更新処理を行う```update```メソッドについてです。
    - ```update```メソッドの引数には```@Valid Diary diary```がありますので、```diary```のフィールドの```id```と```bodyText```に、HTTPのPOSTリクエストされる```id```と```bodyText```を受け取る事ができます。<br>
この変数名の```id```と```bodyText```は、HTMLフォームの```name```属性名と一致させる必要があります。<br>
上で書きました編集画面の```edit.html```の一部を抜粋して確認してみます。
        ```html
        <form th:action="@{/diary/update}" method="post">
            <input type="text" name="bodyText" th:value="${diary.bodyText}">
            <input type="hidden" name="id" th:value="${diary.id}"/>
            ...
        ```
    - ```text```と```hidden```の```input```タグがあり、その2つのデータが```bodyText```と```id```という```name```属性でPOSTリクエストで送信されてくるのがわかります（```id```は編集対象の日記```id```で、```bodyText```は本文です）。<br>
また、引数の```diary```には```@Valid```が付いていますので、```diary```の各フィールドのバリデーションチェック機能が有効になります。<br>
そして、```diary```のバリデーションチェックの結果が、もう一つの引数の```BindingResult bindingResult```に入ります。
    - 最後に渡された```diary```に現在の日時を追加して保存します。（編集時の時間を```CreateDatetime```とする）
        ```java
        diary.setCreateDatetime(LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));
        diaryRepository.save(diary);
        ```
    - ```diary```テーブルのデータ更新処理は、リポジトリインタフェース```DiaryRepository```を使用しています。```save```メソッドは、Spring Data JPAの```ListCrudRepository```が宣言しているメソッドです。
        > ***リポジトリクラスの新規作成処理と更新処理のコードは同じ***<br>
*リポジトリクラス```diaryRepository```の更新処理のコード```diaryRepository.save(diary);```ですが、これは新規投稿時（```add```メソッド）と同じです。*<br>
*```save```メソッドの実装ソースコードを見ると、Databaseのテーブルにレコードが存在するかどうかで実行するSQLを```insert```か```update```で使い分けてくれています。通称```upsert```と言います。*

## 完成！

### お疲れ様でした。これで日記アプリが完成しました。実際に動かしてみて確認してみましょう！
