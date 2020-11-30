# How to create a Bible module for e-Sword

Recently I created a bblx Bible module for [e-Sword](https://www.e-sword.net/). Since there is no documentation for that I decided to share how I did it.

## SQLite
Every bblx file is basically an [SQLite](https://www.sqlite.org/index.html) database file. Once generated, you can upload your bblx file to [SQLite Viewer](https://inloop.github.io/sqlite-viewer/) to see how the file looks like.
An SQLite database file contains one or more tables. For a bblx file you need to have at least two tables. Some bblx files are encrypted, but I didn't find out how to encrypt data for e-Sword.

## Creating SQLite file
For this I used [PHP](https://www.php.net/) programming language. For SQLite manipulation I used the [SQLite3](https://www.php.net/manual/en/book.sqlite3.php) class. So let's start with creating the SQLite file and putting some required information inside.

```PHP
<?php

$db = new SQLite3('kjv.bblx', SQLITE3_OPEN_CREATE | SQLITE3_OPEN_READWRITE);

// create the Details table
$sql = <<<SQL
CREATE TABLE Details (Description NVARCHAR(250), Abbreviation NVARCHAR(50), Information TEXT, Version INT, Font NVARCHAR(50), RightToLeft BOOL, OT BOOL, NT BOOL, Apocrypha BOOL, Strong BOOL)
SQL;

$db->query($sql);

// insert basic information about the module. Version=4 indicates that we are going to use HTML
$sql = <<<SQL
INSERT INTO Details (Description, Abbreviation, Information, Version, Font, RightToLeft, OT, NT, Apocrypha, Strong) VALUES ('King James Version', 'KJV', 'This is the 1769 King James Version of the Holy Bible', 4, 'DEFAULT', 0, 1, 1, 0, 0)
SQL;

$db->query($sql);

// create the Bible table
$sql = <<<SQL
CREATE TABLE Bible (Book INT, Chapter INT, Verse INT, Scripture TEXT)
SQL;

$db->query($sql);

// create an index to speed up e-Sword
$sql = <<<SQL
CREATE INDEX BookChapterVerseIndex ON Bible (Book, Chapter, Verse)
SQL;

$db->query($sql);
```

Now we need to prepare the data we want to import to the database file. If you have a `csv`, `json`, `xml` or any file you can parse it, but I will now use an array to show you how the final data should look like. For verse formatting we will use [HTML](https://www.w3schools.com/html/). You can also use plain text or `rtf`, but `rtf` is much more complicated if you want to style it.

```PHP
$data = [
    [
        'book_id' => 1,
        'chapter_id' => 1,
        'verse_id' => 1,
        'text' => 'In the beginning God created the heaven and the earth.In the beginning God created the heaven and the earth.',
    ],
    [
        'book_id' => 1,
        'chapter_id' => 1,
        'verse_id' => 2,
        'text' => 'And the earth was without form, and void; and darkness <i>was</i> upon the face of the deep. And the Spirit of God moved upon the face of the waters.',
    ],
    // ...
    [
        'book_id' => 66,
        'chapter_id' => 22,
        'verse_id' => 20,
        'text' => 'He which testifieth these things saith, <span style="color: #ff0000">Surely I come quickly.</span> Amen. Even so, come, Lord Jesus.',
    ],
    [
        'book_id' => 66,
        'chapter_id' => 22,
        'verse_id' => 21,
        'text' => 'The grace of our Lord Jesus Christ be with you all. Amen.',
    ],
];
```

Now let's import the verses into the database.

```PHP
$sql = "INSERT INTO Bible(Book, Chapter, Verse, Scripture) VALUES (:book_id, :chapter_id, :verse_id, :text)";

foreach ($data as $verse) {
    $statement = $db->prepare($sql);

    $statement->bindParam(':book_id', $verse['book_id'], SQLITE3_INTEGER);
    $statement->bindParam(':chapter_id', $verse['chapter_id'], SQLITE3_INTEGER);
    $statement->bindParam(':verse_id', $verse['verse_id'], SQLITE3_INTEGER);
    $statement->bindParam(':text', $verse['text'], SQLITE3_TEXT);

    $statement->execute();
}
```

That's it. You can now import the generated bblx module to your e-Sward.
