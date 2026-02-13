```java
// ВОТ СЮДА: вместо checkResult на время
    MvcResult r = MvcTestUtils.performGetOk(mockMvc, "/api/v1/main-nds")
            .andDo(org.springframework.test.web.servlet.result.MockMvcResultHandlers.print())
            .andReturn();

    System.out.println("status=" + r.getResponse().getStatus());
    System.out.println("ct=" + r.getResponse().getContentType());
    System.out.println("body=" + r.getResponse().getContentAsString());

```
