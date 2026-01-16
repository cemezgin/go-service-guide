# Testing Examples

## Mock Generation

```go
// internal/order/usecase/service.go
package usecase

//go:generate mockgen -source=service.go -destination=service_mock.go -package=usecase

type externalClient interface {
    GetOrder(ctx context.Context, orderID, referenceID string) (*model.Order, error)
}

type orderRepository interface {
    Save(ctx context.Context, o *model.Order) error
    FindByOrderID(ctx context.Context, orderID string) (*model.Order, error)
}
```

## Table-Driven Tests

```go
func TestOrderService_SaveOrder(t *testing.T) {
    tests := []struct {
        name               string
        quantity           *string
        expectedType       string
        featureFlagEnabled bool
    }{
        {
            name:               "maps quantity 1 to single when flag ON",
            quantity:           lo.ToPtr("1"),
            expectedType:       "single",
            featureFlagEnabled: true,
        },
        {
            name:               "keeps quantity 1 as-is when flag OFF",
            quantity:           lo.ToPtr("1"),
            expectedType:       "1",
            featureFlagEnabled: false,
        },
        {
            name:               "handles nil quantity",
            quantity:           nil,
            expectedType:       "",
            featureFlagEnabled: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()

            mockRepo := NewMockRepository(ctrl)
            mockClient := NewMockClient(ctrl)
            mockFlags := NewMockFeatureFlags(ctrl)

            mockFlags.EXPECT().
                IsEnabled(gomock.Any(), "quantity_mapping").
                Return(tt.featureFlagEnabled)

            if tt.featureFlagEnabled && tt.quantity != nil {
                mockRepo.EXPECT().
                    Save(gomock.Any(), gomock.Any()).
                    DoAndReturn(func(ctx context.Context, o *Order) error {
                        assert.Equal(t, tt.expectedType, o.Type)
                        return nil
                    })
            }

            svc := NewService(mockRepo, mockClient, mockFlags)
            err := svc.SaveOrder(context.Background(), &Input{Quantity: tt.quantity})

            assert.NoError(t, err)
        })
    }
}
```

## Mock Setup

```go
func TestService_GetOrder(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := mocks.NewMockRepository(ctrl)
    mockClient := mocks.NewMockClient(ctrl)

    // Setup expectations
    mockRepo.EXPECT().
        FindByID(gomock.Any(), "order-123").
        Return(&Order{ID: "order-123", Status: "active"}, nil)

    mockClient.EXPECT().
        Validate(gomock.Any(), gomock.Any()).
        Return(true, nil).
        Times(1)

    // Create service with mocks
    svc := NewService(mockRepo, mockClient)

    // Execute
    result, err := svc.GetOrder(context.Background(), "order-123")

    // Assert
    assert.NoError(t, err)
    assert.Equal(t, "order-123", result.ID)
    assert.Equal(t, "active", result.Status)
}
```

### Mock Matchers

```go
// Any value
mockRepo.EXPECT().Save(gomock.Any(), gomock.Any())

// Specific value
mockRepo.EXPECT().FindByID(gomock.Any(), "specific-id")

// Custom matcher
mockRepo.EXPECT().Save(gomock.Any(), gomock.Cond(func(o *Order) bool {
    return o.Status == "active"
}))
```

### Mock Return Values

```go
// Return values
mockRepo.EXPECT().FindByID(gomock.Any(), "id").Return(&Order{}, nil)

// Return error
mockRepo.EXPECT().FindByID(gomock.Any(), "id").Return(nil, errors.New("not found"))

// Dynamic return
mockRepo.EXPECT().Save(gomock.Any(), gomock.Any()).DoAndReturn(
    func(ctx context.Context, o *Order) error {
        o.ID = "generated-id"
        return nil
    },
)
```

## Assertions

```go
func TestOrderValidation(t *testing.T) {
    t.Run("valid order", func(t *testing.T) {
        order := &Order{Number: "123", Reference: "ABC"}
        err := order.Validate()

        assert.NoError(t, err)
        assert.NotNil(t, order)
    })

    t.Run("invalid order", func(t *testing.T) {
        order := &Order{Number: "", Reference: ""}
        err := order.Validate()

        assert.Error(t, err)
        assert.Contains(t, err.Error(), "number is required")
    })

    t.Run("slice assertions", func(t *testing.T) {
        items := []string{"a", "b", "c"}

        assert.Len(t, items, 3)
        assert.Contains(t, items, "b")
        assert.NotContains(t, items, "d")
    })

    t.Run("error type assertions", func(t *testing.T) {
        err := NewNotFoundError("order-123")

        assert.True(t, errors.Is(err, &NotFoundError{}))
    })
}
```

## Integration Tests

```go
func TestIntegrationOrderFlow(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    // Setup real dependencies
    db := setupTestDB(t)
    defer db.Close()

    repo := repository.New(db)
    client := client.New(testConfig)
    svc := usecase.NewService(repo, client)

    // Test full flow
    ctx := context.Background()

    // Create
    order := &model.Order{Number: "TEST-123", Reference: "REF-456"}
    err := svc.Create(ctx, order)
    assert.NoError(t, err)

    // Read
    found, err := svc.GetByNumber(ctx, "TEST-123")
    assert.NoError(t, err)
    assert.Equal(t, order.Number, found.Number)

    // Cleanup
    err = svc.Delete(ctx, order.ID)
    assert.NoError(t, err)
}
```
