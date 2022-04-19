---
layout: post
title: "[REACT] Wrapping custom component with Ant design's &lt;Form.Item&gt;"
categories: [Development]
tags: [frontend, react, typescript, antd, ant-design, form, form-item]
feature_image: '/assets/img/20220419/feature.jpg'
feature_license: 'Photo by <a href="https://unsplash.com/@flowforfrank?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Ferenc Almasi</a> on <a href="https://unsplash.com/s/photos/react-antd?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>'
use_math: false
published: true
last_modified_at: '2022-04-19 15:03:58'
---

<!-- more -->
# React + Typescript 에서 Custom component 를 Ant-Design 의 &lt;Form.Item&gt; component 로 wrapping 하여 사용하기

## Simple form control

`<Input />` 같은 단순한 form control 의 경우 그냥 `<Form.Item>` 으로 감싸면서 `name` property 를 주면 submit handler 에서 parameter 로 바로 받을 수 있다.

```javascript
const onSubmit = (values: any) => {
  console.log(values);
  // {text: 'test'}
};

<Form onFinish={onSubmit}>
  <Form.Item label="Text" name="text">
    <Input />
  </Form.Item>
</Form>
```

## Custom component wrapped by &lt;Form.Item&gt;

직접 만든 component 를 form item 으로 사용하고 싶다면 해당 component 의 내부에 `onChange` property 를 설정해 주면 된다.

예시)
- `TagInput`
  - Input textbox 에서 enter 를 누를 때마다 tag 를 추가하는 component
  - 자체 state 로 등록된 tag list 를 유지한다.
  - Form 의 Submit 버튼을 클릭하면 유지하고 있던 tag list 를 form 에 제출한다.

> TagInput.tsx

```typescript
import { useState } from 'react';
import { Tag, Input } from 'antd';

export interface TagInputProps {
  values?: string[];
  // onChange => name property 가 지정된 <Form.Item> 으로 감쌀 경우,
  // child 의 onChange property 는 자동으로 form control 에 등록된다.
  onChange?: (taglist: string[]) => void;
}

export function TagInput(props: TagInputProps) {
  // text input value
  const [inputValue, setInputValue] = useState<string>('');
  // tag list
  const [tags, setTags] = useState<string[]>(props.values ?? []);

  const handleEnter = () => {
    let tag: string = inputValue;
    if (tag !== '' && !tags.includes(tag) && tags.length < 5) {
      let update = [...tags, tag];
      setTags(update);
      setInputValue('');
      if (props.onChange != null) {
        props.onChange(update);
      }
    }
  };

  const handleRemoveTag = (removedTag: string) => {
    let update = tags.filter((tag) => tag !== removedTag);
    setTags(update);
    if (props.onChange != null) {
      props.onChange(update);
    }
  };

  return (
    <>
      <Input
        placeholder="New tag"
        onChange={(e) => setInputValue(e.target.value)}
        onPressEnter={handleEnter}
        value={inputValue}
      />
      {tags.map((tag, index) => {
        return (
          <Tag
            key={tag}
            closable={true}
            onClose={() => handleRemoveTag(tag)}
          >
            {tag}
          </Tag>
        );
      })}
    </>
  );
}
```

[API Document](https://ant.design/components/form/#Form.Item) 를 참고하면,
Antd 에서 `<Form.Item>` 으로 감싼 child 의 `onChange` prop 은 자동으로 form control 에 등록된다.
실제로는 위와 같이 `onChange` property 를 nullable 로 선언해 두고, `<Form.Item>` 으로 감쌀 때 이 프로퍼티를 명시하지 않으면 Antd 가 자체적으로 default function 을 injection 하는 것 같다.

단순히 `onChange` property 를 설정하는 것 만으로는 submit 시에 값이 올라오지 않으며, `handleEnter`, `handleRemoveTag` 에서처럼 form 에 제출할 값에 변화가 생길 때마다 `props.onChange()` 를 호출해 줘야 값이 제대로 올라온다.

추가로 이 상태에서는 tag 를 추가하기 위헤 enter 를 입력할 경우 이 이벤트에 `TagInput` 과 `Form` 이 둘 다 반응하므로 [Tag 추가] 와 [Submit] 동작이 동시에 일어난다. 이를 방지하기 위해 `<Form>` tag 에 

```javascript
<Form
  ...
  onKeyDown={(e) => {
    if (e.key === 'Enter') {
      e.preventDefault();
    }
  }}
>
```

enter 키 입력을 무시하도록 설정해주면 된다.