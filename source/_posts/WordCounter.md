---
title: WordCounter
date: 2022-03-18 10:45:31
tags: Design
---

```
import java.util.stream.IntStream;
import java.util.stream.Stream;

/**
 * 计算单词个数
 * <U> U reduce(U identity,
 * BiFunction<U, ? super T, U> accumulator,
 * BinaryOperator<U> combiner);
 */
public class WordCounter {
    private final boolean lastSpace;
    private final int count;

    public WordCounter(boolean lastSpace, int count) {
        this.lastSpace = lastSpace;
        this.count = count;
    }

    public WordCounter accumulator(Character character) {
        if (Character.isWhitespace(character)) {
            return lastSpace ? this : new WordCounter(true, count);
        } else {
            return lastSpace ? new WordCounter(false, count + 1) : this;
        }
    }

    public WordCounter combiner(WordCounter wordCounter) {
        return new WordCounter(lastSpace, count + wordCounter.count);
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args) {
        String str = " I am a boy ";
        Stream<Character> characterStream = IntStream.range(0, str.length()).mapToObj(str::charAt);
        WordCounter wordCounter =
                characterStream.reduce(
                        new WordCounter(true, 0), WordCounter::accumulator, WordCounter::combiner);
        System.out.println(wordCounter.getCount());
        System.out.println(countWordsIteratively(str));

    }

    public static int countWordsIteratively(String str) {
        boolean lastSpace = true;
        int count = 0;
        char[] chars = str.toCharArray();
        for (int i = 0; i < chars.length; i++) {
            if (Character.isWhitespace(chars[i])) {
                lastSpace = true;
            } else {
                if (lastSpace) count++;
                lastSpace = false;
            }
        }
        return count;
    }
}

```

