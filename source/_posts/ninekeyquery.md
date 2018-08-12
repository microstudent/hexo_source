---
title: 实现一个简单的九键拼音匹配算法(比较随性)
date: 2016-5-29 20:27:43
tags: cacw
---

好久没写博文了，因为最近都在忙课设的事情。

说起课设，我最近在做的一个APP有一个需求，是做一个类似系统拨号软件的功能，可以实现9键拼音筛选列表。话不多说，看图：

![image](https://raw.githubusercontent.com/microstudent/microstudent.github.io1/master/ninekey.gif)
<!-- more -->
需求是对拨号窗口的九键进行拼音筛选，需要设计一种算法来筛选条目。
应用类似手机的通讯录。

方法给出的queryString为一串数字，通过拨号键盘给出的。也可能掺杂着#*等电话符号。
假设User类有getPinyin()和getPhone()两个方法，分别得到格式为String[]的拼音和String的电话号码。

经过一番尝试，我的拼音算法的最终实现代码如下：
```
public class NineKeyQuery implements IQuery {
    private static final String[] NINE_KEY_WORD = {"", "", "ABC", "DEF", "GHI", "JKL", "MNO", "PQRS", "TUV", "WXYZ"};
    private final List<IUser> filteredModelListByNumber = new ArrayList<>();
    private final List<IUser> filteredModelListByName = new ArrayList<>();
    private final Set<IUser> result = new LinkedHashSet<>();

    @Override
    public List<? extends IUser> filter(List<? extends IUser> data, String queryString) {
        if (queryString.isEmpty()) {
            return data;
        }
        result.clear();
        result.addAll(queryByNumber(data, queryString));
        result.addAll(queryByName(data, queryString));
        return new ArrayList<>(result);
    }

    private List<IUser> queryByName(List<? extends IUser> data, String queryString) {
        filteredModelListByName.clear();
        if (data != null) {
            for(int i = 0; i < data.size();i++){
                //取出拼音
                String[] pinyin = PinyinUtils.getPinyinString(data.get(i).getName());

                //分别为当前正在检索的字的index，和该词的pinyin的offset,对每个User分别从第0个的第0位开始查询
                int wordNow = 0, pinyinOffset = 0;
                boolean flag = false;
                for (int n = 0; n < queryString.length(); n++) {
                    char number = queryString.charAt(n);

                    //对查询语句的每一个字，包括*#等符号
                    String keywords = getKeywords(number);//提取出该数字代表的字母，例如1代表abc
                    if (keywords != null && !keywords.isEmpty() && !pinyin[wordNow].isEmpty()) {//合法性检查
                        for (int j = 0; j < keywords.length(); j++) {//对当前的查询语句的某一个字的要素进行检查，只要有匹配的即可进入下一个查询字。
                            //1. 先对wordNow进行合法性检查，如果合法性不合格直接为false退出循环
                            if (wordNow >= pinyin.length) {
                                flag = false;
                                break;
                            }
                            //2. 检查当前正在检索的字的offset所在的字母符不符合，符合设Flag为true退出循环
                            if (pinyinOffset < pinyin[wordNow].length() && pinyin[wordNow].charAt(pinyinOffset) == keywords.charAt(j)) {
                                flag = true;
                                pinyinOffset++;
                                break;
                            }
                            //3. 再检查下一个字首字母是否符合要求，如果符合设true退出循环,注意合法性检查
                            if (pinyinOffset != 0 && wordNow < pinyin.length - 1 && !pinyin[wordNow + 1].isEmpty() && pinyin[wordNow + 1].charAt(0) == keywords.charAt(j)) {
                                flag = true;
                                wordNow++;
                                pinyinOffset = 1;
                                break;
                            }
                            //4. 如果前面的检查都失败了，继续下一个检查元素
                            flag = false;
                        }
                    }

                    //这里检查flag是否为false，为false直接退出循环，因为已经都不匹配，为true则继续检查
                    if (!flag)
                        break;
                }
                if(flag)
                    filteredModelListByName.add(data.get(i));
            }
        }
        return filteredModelListByName;
    }

    private String getKeywords(char number) {
        if (number <= '9' && number >= '0') {
            return NINE_KEY_WORD[number - '0'];
        } else return "";
    }


    private List<IUser> queryByNumber(List<? extends IUser> data, String queryString) {
        filteredModelListByNumber.clear();
        if (data != null) {
            for (IUser user : data) {
                if (user.getPhone().contains(queryString)) {
                    filteredModelListByNumber.add(user);
                }
            }
        }
        return filteredModelListByNumber;
    }
}
```

有兴趣的同学可以去[这里看看](https://github.com/fivefire/GDUTContacts/blob/master/app/src/main/java/com/fivefire/app/gdutcontacts/widget/dialpad/query/NineKeyQuery.java)，这是一个最近在做的课设，里面只有那个九键键盘是我做的:)

之前还做了一个有动画效果的快速滚动条，和三星的滚动条类似，有兴趣也可以去[这里](https://github.com/microstudent/BouncyScroller)逛逛！