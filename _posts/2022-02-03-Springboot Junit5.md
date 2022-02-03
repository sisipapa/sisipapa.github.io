---
layout: post
title: Springboot Junit5
category: [junit5]
tags: [springboot, junit5]
redirect_from:

- /2022/02/03/

---

## Springboot Junit5 단위테스트
그동안 Controller, Service, Repository(mapper) 단위테스트를 할 때 SpringBootTest 어노테이션을 붙여서 단위테스트를 진행했었다. 최근에 TDD에 대한 정리를 하다가 Junit5에 대한 정리를 해봐야 겠다는 생각을 하게 되었고 오랜만에 노트를 작성하게 되었다. 오늘은 Controller, Service, Repository 별로 단위테스트를 위한 나만의 최적화를 찾아보려고 한다.  

## JPA Repository Junit5 TEST CASE
```java
import com.sisipapa.junit.domain.Item;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

//@TestPropertySource(locations = "classpath:application-test.yml")
//@ActiveProfiles("test")
//@DataJpaTest
@ExtendWith(MockitoExtension.class)
class ItemRepositoryTest {

//    @Autowired
//    private ItemRepository itemRepository;

    //    @MockBean
    @Mock
    private ItemRepository itemRepository;


    @Test
    void save(){
        // given
        final Item item = Item.builder().name("item1").description("아이템1입니다.").build();

        when(itemRepository.save(any())).thenReturn(item);

        // when
        final Item saveItem = itemRepository.save(new Item());

        // then
        verify(itemRepository).save(any());
        assertEquals(saveItem.getName(), "item1");
        assertEquals(saveItem.getDescription(), "아이템1입니다.");
    }
}
```   

## Mapper Junit 테스트
```java
import com.sisipapa.junit.domain.dto.ProductDto;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.mockito.junit.jupiter.MockitoExtension;
import org.mybatis.spring.boot.test.autoconfigure.MybatisTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ProductMapperTest {

    @Mock
    private ProductMapper productMapper;


    @BeforeEach
    void setUp() {
    }

    @Test
    void findAll() {
        // given
        List<ProductDto> mockProducts = List.of(
                ProductDto.of().idx(1L).name("베베숲 물티슈").description("베베숲 물티슈 설명").build()
                ,ProductDto.of().idx(2L).name("여름 토퍼").description("여름 토퍼 설명").build()
                ,ProductDto.of().idx(3L).name("페이크 삭스").description("페이크 삭스 설명").build()
                ,ProductDto.of().idx(4L).name("우산").description("우산 설명").build()
        );
        when(productMapper.findAll()).thenReturn(mockProducts);

        // when
        List<ProductDto> resultProducts = productMapper.findAll();

        // then
        verify(productMapper).findAll();
        assertThat(mockProducts.get(0).getName()).isEqualTo(resultProducts.get(0).getName());
    }

    @Test
    void findByName() {
        // given
        String name = "베베숲 물티슈";
        when(productMapper.findByName(name)).thenReturn(ProductDto.of().idx(1L).name("베베숲 물티슈").description("베베숲 물티슈 설명").build());

        // when
        ProductDto mockProductDto = productMapper.findByName(name);

        //then
        verify(productMapper).findByName(name);
        assertThat(mockProductDto.getName()).isEqualTo("베베숲 물티슈");
        assertThat(mockProductDto.getDescription()).isEqualTo("베베숲 물티슈 설명");

    }

    @Test
    void insertProduct() {
        // given
        ProductDto dto = ProductDto.of().name("신규1").description("신규1 설명").build();
        when(productMapper.insertProduct(dto)).thenReturn(1);

        // when
        int cnt = productMapper.insertProduct(dto);

        // then
        verify(productMapper).insertProduct(dto);
        assertThat(cnt).isEqualTo(1);
    }

    @Test
    void updateProduct() {
        // given
        ProductDto dto = ProductDto.of().idx(1L).name("베베숲 물티슈 >> 수정").description("베베숲 물티슈 설명 >> 수정").build();
        when(productMapper.updateProduct(dto)).thenReturn(1);

        // when
        int cnt = productMapper.updateProduct(dto);

        // then
        verify(productMapper).updateProduct(dto);
        assertThat(cnt).isEqualTo(1);
    }

    @Test
    void deleteProduct() {
        // given
        Long idx = 1L;
        when(productMapper.deleteProduct(idx)).thenReturn(1);

        // when
        int cnt = productMapper.deleteProduct(idx);

        // then
        verify(productMapper).deleteProduct(idx);
        assertThat(cnt).isEqualTo(1);
    }
}
```  

### @ExtendWith(MockitoExtension.class)  
- Spring에서 Mockito를 사용하기 위해서는 @ExtendWith(MockitoExtention.class) 를 테스트 클래스 상단에 지정해주면 된다.   
- @Mock 어노테이션을 통해 간단하게 Mock 객체를 만들어 사용할 수 있다.  

### @DataJpaTest  
- @DataJpaTest 어노테이션은 Full-Auto Config를 하지 않고 JPA 관련 테스트 설정을 로드한다.   
- @DataJpaTest는 기본적으로 @Entity 어노테이션이 적용된 클래스를 스캔하여 스프링 데이터 JPA 저장소를 구성한다.      

### @Mock  
- Mock은 가짜객체를 만드는데 스프링빈에 등록이 안되는 객체이다.   
- 스프링 컨테이너가 DI를 하는 방식이 아니라 객체생성시 생성자에 Mock객체를 직접 주입한다.   
- 스프링을 띄우지 않으므로 MockBean을 사용할때보다 빠르다.  

### @MockBean  
- MockBean은 가짜 Bean을 스프링에 등록된다.   
- 스프링 컨테이너가 기존에 갖고있는 Bean객체는 MockBean객체로 치환되어 DI된다.  

### when~thenReturn VS given~willReturn  
- BDDMockito가 제공하는 기능과 Mockito가 제공하는 기능은 별반 다르지 않다.   
- BDD라는 것을 테스트 코드에 도입할 때 기존의 Mockito가 가독성을 해치기 때문에 이를 해결하기 위한 기능은 같지만 이름만 다른 클래스라고 생각해도 될 것 같다.  

### Mockito.verify
- verify(mock).method(param) : 해당 Mock Object 의 메소드를 호출했는지 검증 ex) verify(mock).get(0);  
- verify(mock, times(wantedNumberOfInvocations)).method(param) : 해당 Mock Object 의 메소드가 정해진 횟수만큼 호출됬는지 검증 ex) verify(mock, times(2)).get(anyInt());  
- verify(mock, atLeast(minNumberOfInvocations)).method(param) : 해당 Mock Object 의 메소드가 최소 정해진 횟수만큼 호출됬는지 검증 ex) verify(mock, atLeast(1)).get(anyInt());  
- verify(mock, atLeastOnce()).method(param) : 해당 Mock Object 의 메소드가 최소 한번 호출됬는지 검증 ex) verify(mock, atLeastOnce()).get(anyInt());  
- verify(mock, atMost(maxNumberOfInvocations)).method(param) : 해당 Mock Object 의 메소드가 정해진 횟수보다 적게 호출됬는지 검증 ex) verify(mock, atMost(2)).get(anyInt());  
- verify(mock, never()).method(param) : 해당 Mock Object 의 메소드가 호출이 안됬는지 검증 ex) verify(mock, never()).get(2);

> JPA에서 실제 데이터베이스와 연결해서 단위테스트를 해보고 싶은 경우 @DataJpaTest @Autowired private ItemRepository itemRepository 주석을 풀고 적용할 수 있다.   




## 참고  
[Junit5(Juptier) Service 테스트코드 작성법(Mock, MockBean 차이점 확인)](https://frozenpond.tistory.com/83)
[Mockito와 BDDMockito는 뭐가 다를까?](https://velog.io/@lxxjn0/Mockito%EC%99%80-BDDMockito%EB%8A%94-%EB%AD%90%EA%B0%80-%EB%8B%A4%EB%A5%BC%EA%B9%8C)
[[JUnit & Mockito] Verify Method Calls](https://velog.io/@dnjscksdn98/JUnit-Mockito-Verify-Method-Calls)

## Github
<https://github.com/sisipapa/junit5.git>  




