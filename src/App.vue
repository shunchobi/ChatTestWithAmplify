<template>
  <template v-for="data, index in chatMessage" :key="index">
    <h3>{{ data.message }}</h3>
  </template>
  <div style="margin: 50px;"></div>
  <input v-model="inputMessage" type="text">
  <button @click="saveData">Save</button>
</template>

<script setup lang="ts">
import { Amplify, API, graphqlOperation } from 'aws-amplify';
import { onMounted, onUnmounted, ref } from 'vue';
import { ListChatMessagesQuery, OnCreateChatMessageSubscription } from './API';
import awsExports from './aws-exports';
import { createChatMessage } from './graphql/mutations';
import { listChatMessages } from './graphql/queries';
import { onCreateChatMessage } from './graphql/subscriptions';
import { ChatMessage } from './models';

Amplify.configure(awsExports);  
type SubscriptionEvent = { value: { data: OnCreateChatMessageSubscription } };


const inputMessage = ref<string>('');
const chatMessage = ref<ChatMessage[]>([]);
// unsubscribe(購読解除)で型エラーが出るので今回はanyで受ける
const onCreate: any = API.graphql(graphqlOperation(onCreateChatMessage));


const setChatMessages = (value: ChatMessage[]) => {
  chatMessage.value = value;
}

const saveData = async () => {
  const model = new ChatMessage({
    message: inputMessage.value!,
  });

  await API.graphql(
    graphqlOperation(createChatMessage, {
      input: model,
    })
  );

  inputMessage.value = '';
}

const fetchData = async () => {
  const items = await API.graphql(graphqlOperation(listChatMessages));
  
  if ("data" in items && items.data) {
    const messages = items.data as ListChatMessagesQuery;
    setChatMessages(
      sortMessage(messages.listChatMessages?.items as unknown as ChatMessage[])
    );
  }
}

onCreate.subscribe({
  next: ({ value: { data } }: SubscriptionEvent) => {
    // 更新されたデータをリストに追加
    const newMessage: ChatMessage = data.onCreateChatMessage! as unknown as ChatMessage;
    setChatMessages(
      sortMessage([...chatMessage.value, newMessage]) 
    );
  },
});


const sortMessage = (messages: ChatMessage[]) => {
  return [...messages].sort(
    (a, b) =>
      new Date(a.createdAt!).getTime() - new Date(b.createdAt!).getTime()
  );
}


onMounted(() => {
  fetchData();
});

onUnmounted(() => {
  // 画面を離れる時に購読解除する
  // SPA(シングルページアプリケーション)で、HTMLの再読み込み無しに画面を行ったり来たりすると購読処理が何度も行われてしまいます。
  // 1回のデータ登録に対して複数回の通知が来るので、データを重複処理してしまう危険性があります。
  // ですので画面を離れる時など、購読が不要になるタイミングで解除するようにしましょう。
  onCreate.unsubscribe();
});

</script>

<style scoped>  

</style>